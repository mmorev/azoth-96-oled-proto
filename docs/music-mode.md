# GearLink Music Mode — реверс компоненты и USB-протокола (ASUS ROG Azoth 96 HE)

**Дата:** 2026-10-03 · **Метод:** декомпиляция .NET-компонент (ILSpy/Ghidra) + живые USB-захваты USBPcap во время активного music mode
**Устройство:** VID `0x0B05` PID `0x1C10` (адрес на шине в захватах 13–15: **2**; меняется при переподключении)
**Канал OLED:** интерфейс 1, EP 0x01 OUT / 0x82 IN, репорты 64 байта без report ID (см. `report.md`)

Базовый протокол (карта устройства, опкоды 0x12/0x63/0x66/0x6A/0x6B/0x61 и пр.) — в `report.md`.
Этот документ — про music mode: эквалайзер + информация о треке.

---

## 1. Компоненты GearLink (задачи планировщика `\ASUS\`, запуск при логине)

| Задача | Бинарник | Роль в OLED |
|---|---|---|
| `GearLink_KBAgentTask` | `C:\Program Files (x86)\ASUS\Gear Link\KB\GearLink_KBAgent\GearLink_KBAgent.exe` | **Главный: music mode.** BASS (`bass.dll`, `basswasapi.dll`, `Bass.Net.dll`) — WASAPI loopback + FFT; `MusicInfo` — чтение SMTC (Windows.Media.Control); рендер текста в PNG → RGB565; `HWMonitor` (OPHWInfo/cpuid) — метрики CPU/temp для опкода `0x66`; `WinNotification`. **TCP-сервер 127.0.0.1:8300** (порт в `HKCU\Software\ASUS\Gear Link\KB\KBAgent_Device_Port`) |
| `GearLink_UtilityCompanionTask` | `...\Gear Link\GearLink_UtilityCompanion\GearLink_UtilityCompanion.exe` | Хост UI; грузит `GearLink_KBApi.dll` — **HID-клиент**: берёт JSON от KBAgent и шлёт опкод `0x67` в клавиатуру |
| `GearLink_KBServiceTask` | `...\KB\GearLink_KBService\GearLink_KBService.exe` | Брокер: TCP 8000 (HTML UI), 8100 (device API) |
| `GearLink_MacroServerTask` | `...\GearLink_UtilityCompanion\GearLink_MacroServer.exe` | Макросы (OLED не касается) |
| `GearLink_PowerNotificationTask` | `...\GearLink_UtilityCompanion\GearLink_PowerNotification.exe` | Батарея/питание (не OLED) |
| `\ASUS\ArmourySocketServer` | ArmouryDevice\...\TaskSchedulerTool_ArmourySocketServer.exe | Общая инфраструктура Armoury Crate |

Декомпилированные исходники: `analysis/decomp_gearlink/{kbagent,kbapi,kbservice}`.

## 2. Конвейер music mode

```
[Аудио] ──WASAPI loopback──> BASS FFT ──> 32 значения 0..255 (лог-бины)
                                              │ JSON {class_id:10042000, spectrum_data}
[SMTC] ──title/artist/album──> GDI-рендер ──> PNG 48×N ──> RGB565 LE (.bin)
                                              │ JSON {class_id:10032000, binary_path}
              ┌───────── GearLink_KBAgent.exe (TCP 8300, push) ────────┐
              ▼
   GearLink_KBApi.dll (в UtilityCompanion) — MusicMode_RGB_Type_1
              │ HID write, opcode 0x67
              ▼
        Клавиатура: сама рисует бары эквалайзера + композитит инфо-битмап
```

- KBAgent **ничего не знает про USB** — он рендерит/считает и шлёт JSON по TCP.
- KBApi **ничего не считает** — берёт `MusicInfo.bin` с диска и льёт чанками в HID.
- Azoth 96 HE использует класс `MusicMode_RGB_Type_1` (`class_id_parent 50050001`), а не GRAY-вариант
  (GRAY — для других моделей: frame 208×64, интервал спектра 80 мс, ACK на каждый чанк).

## 3. JSON-API KBAgent (TCP 127.0.0.1:8300)

Строки `{...}` (JSON-объекты, склейка по сбалансированным скобкам). Два сокета: cmd + stream.
Ответы/пуши идут обратно тем же сокетом. Ключевые `class_id` (из `ClassID_GearLink_KBAgent.cs`):

| class_id | Класс | Смысл |
|---|---|---|
| 10010000/10050000 | ConvertImageToBinary RGB/GRAY | PNG → сырой RGB565 / gray |
| 10020000/10060000 | SaveFontAsImage RGB/GRAY | Текст → PNG (шрифт, поворот) |
| 10030000 | ConvertMusicInfoToBinary RGB | **Инфо о треке**: поллинг SMTC, рендер, push |
| 10040000 | MusicSpectrum | **Спектр**: WASAPI FFT, push каждые `interval` мс |
| 10080000 | HWMonitor | CPU/GPU/RAM/temp (источник опкода `0x66`) |
| 10090000 | WinNotification | Уведомления Windows на OLED |
| 10100000/1011xxxx | GamingMode / LaunchApp* | Прочее |

Команда спектра (KBApi → KBAgent): `{"vid":"2821","pid":"7184","sn":"...","class_id":"10040000","class_id_parent":"50050001","interval":"60","spectrum_lines":"32","on_off":"1"}`.
Push спектра обратно: `{..., "class_id":"10042000","spectrum_data":"<HtmlEncode(UTF8-строка байтов)>"}`.
Команда инфо (period): `{"class_id":"10030000","class_id_parent":"50050001","frame_width":"136","frame_height":"48","direction":"1","font_name":"...","font_style":"0","font_size":"14","width_offset":"6","height_offset":"6","max_width":"680","max_height":"48","empty_buffer_x":"5","empty_buffer_y":"5","image_path":"...MusicInfo.png","on_off":"1","interval":"3000"}`.

Формула спектра (`MusicSpectrum.cs`): бин-границы `2^(10·j/(lines−1))` (лог-шкала, j=0..31),
в группе берётся **максимум** амплитуды, значение = `clamp(sqrt(a)·3·255 − 4, 0, 255)`.

## 4. HID-протокол music mode — опкод `0x67` (103)

Wire-формат: 64-байтный репорт в EP 0x01, байт 0 = `0x67`, байт 1 = субкоманда.
(`0x67` = `array[1]` в коде при report_id=0; код в `MusicMode_RGB_Type_1.cs`.)

### 4.1 `67 00` — SetMusicMode (инициализация)

```
байт:  0     1    2   3 4       5      6       7 8        9 10     11 12
знач:  0x67  0x00 seq=0 u16  mode   style   total u16LE  w u16LE  h u16LE
```
- `mode`: 0 = только инфо, 1 = только спектр, 2 = инфо сверху + спектр снизу (текущий профиль),
  3 = спектр сверху + инфо снизу
- `style`: стиль спектра (в профиле = 1)
- `total`: число чанков инфо-битмапа (0, если mode=1)
- `w`, `h`: **ландшафтные** размеры области инфо (напр. 136×48, 177×48, 268×48; высота всегда 48)
- ACK: эхо `67 00 ...` на EP 0x82 (устройство отвечает ~0.2 мс)

### 4.2 `67 01` — чанк инфо-битмапа

```
67 01 [idx u16 LE] [60 байт данных, паддинг 00]
```
- **Порядок: idx от total−1 вниз до 0.**
- Чанк с индексом `idx` несёт байты буфера `[(total−1−idx)·60 … +60)` — т.е. idx = остаток (remaining).
- Последний чанк (idx=0) — хвост + паддинг.
- **ACK пачками**: устройство эхо-подтверждает раз в ~20 чанков (~21 мс): `67 01 [idx u16 LE] 00 00…`.
  Хост стримит без ожидания: **~1.0 мс/чанк (~960 чанков/с, ≈57 КБ/с)**.
- При потере чанка устройство шлёт `67 01 FF AA …` — запрос ретрансмиссии (в захватах не встречался).

### 4.3 `67 02` — кадр спектра (эквалайзер)

```
67 02 [seq u16 = 0] [32 байта: высоты колонок 0..255] [паддинг 00 до 64]
```
- GearLink шлёт **32 байта** каждые **60 мс** (замер по дампу: среднее 63.3 мс, 15.8 Гц, min 50 / max 111).
- В коде лимит — `min(len,60)`: до 60 колонок, но зашитая конфигурация = 32.
- ACK нет. Дедупликация: кадр, равный предыдущему, не отправляется.
- Тишина: хост сам шлёт затухающие кадры (×0.8 за тик), пока бары не упадут до ~0, потом перестаёт
  слать (в дампе: пауза музыки → плавное падение ~4 с → полная тишина в USB).

### 4.4 `67 03` — ShowOnly (перерисовать без данных)

`67 03 [seq u16=0]`, ожидает ACK (`m_hidResponse.Add(1002)`).

## 5. Инфо-битмап: формат (подтверждено декодированием)

- Рендер: текст `title\rartist\ralbum`, шрифт из профиля (сейчас 微軟正黑體 14), повёрнут на 90°.
- **Пиксельный буфер — портретный растр: ширина 48 колонок × N строк**, row-major, **RGB565 little-endian**
  (`r = v>>11`, `g = (v>>5)&0x3F`, `b = v&0x1F`, каждый сдвинут <<3/<<2/<<3).
- Поворот декода на 90° по часовой → читаемый ландшафтный кадр `N×48`.
- N = ширина текста в ландшафте: 136 (короткий трек) … max_width=680 (длинный, marquee).
- Примеры размеров: 48×136 → 13056 B; 48×177 → 16992 B (total=284); 48×268 → 25728 B.
- **Скролл длинных названий делает УСТРОЙСТВО**: при прокрутке 268-px баннера в окне 136 px
  в USB нет ни одной перезаливки (25 с захвата — только спектр).

### Пруф (полный цикл захвачен и декодирован)

Захват `13-music-mode.pcap`, t=90.8 с: `67 00 …` + 284 чанка `67 01` за **297 мс** →
собрано `analysis/reassemble_capture.py` → декод `decode565.ps1` → читаемый текст
**«Walk The Walk / Gaz Coombes»** (трек, игравший в момент захвата; совпал с отображением на OLED).
Аналогично декодированы живые `MusicInfo.bin` с диска: «Dead Man / The Parlor Mob», «When You Cry /
Tito & Tarantula» (совпало с текущим SMTC-сеансом).

## 6. Триггеры и живые конфиги

- **Спектр**: стартует по команде `10040000` c `on_off=1`; тикает в KBAgent каждые `interval` мс.
- **Инфо**: поллинг SMTC каждые **3000 мс**; при изменении строки `title\rartist\ralbum` — рендер +
  полный ре-аплоад (`67 00` + все чанки `67 01`). Пауза/смена статуса без смены текста — трафика нет.
- Профиль (UI-настройки): `%LOCALAPPDATA%\ASUS\Gear Link\KB\ROG AZOTH 96 HE\<SN>\OLED\Profile 1\MusicMode_RGB_Type_1.txt`
  ```json
  {"class_id":"50050001","profile_index":"1","music_mode":"2","spectrum_style":"1",
   "font_name":"%E5%BE%AE%E8%BB%9F%E6%AD%A3%E9%BB%94","font_style":"0","font_size":"14",
   "on_off":"1","vid":"2821","pid":"7184","sn":"W5MPKR087113"}
  ```
- Рендер текущего трека: `%LOCALAPPDATA%\ASUS\Gear Link\KB\7184\<SN>\MusicInfo\MusicInfo.png` (RGBA)
  и `MusicInfo.bin` (сырой RGB565 — то, что уходит в USB).
- Порты: `HKCU\Software\ASUS\Gear Link\KB` → `KBAgent_Device_Port`=8300, `KBService_Device_Port`=8100,
  `KBService_HTML_Port`=8000.

## 7. Сравнение каналов загрузки контента (music vs fullscreen GIF vs banner)

В прошивке **три разных канала** загрузки графики — общая идеология (чанковый стрим + пакетные ACK +
`FF AA`-ретрансмиссия), но разные опкоды, раскладки пакетов и форматы бинарников:

| | **Music info** | **Custom Animation (картинка/GIF, полный экран)** | **Custom Banner (бегущий текст)** |
|---|---|---|---|
| Класс KBApi | MusicMode_RGB_Type_1 (50050001) | CustomAnimation_RGB_Type_1 (50030001) | CustomBanner_RGB_Type_1 (50040001) |
| HID опкод | `0x67` | `0x61` | `0x62` |
| Init | `67 00 [seq u16][mode][style][total u16][w u16][h u16]` — геометрия переменная | `61 01 [seq u16][total u32]` — **без w/h: геометрия фиксирована панелью** | `62 00 …` |
| Чанк | `67 01 [idx u16][60 Б]` | `61 02 [idx u32][58 Б]` | `62 01 [idx u16][60 Б]` |
| Показ | `67 03` | `61 03` | `62 02` |
| Бинарник | ONLY_DATA: сырые пиксели без заголовка | **STANDARD: `[frames u16][duration u16 × frames]` + кадры** | ONLY_DATA |
| Геометрия | 48 × N (N=136…680), портретный буфер, RGB565 LE | кадр 97×184 (184×97 ланд.), статика вращается хостом на 90° CW; **кадры GIF не вращаются** (готовить в ориентации панели) | 85 × N (frame 136×85), портретный |
| Анимация | 1 кадр, замена при смене трека | **GIF → N кадров с индивидуальными длительностями, устройство крутит цикл само** (хост больше ничего не шлёт) | скролл на устройстве |
| Статика | — | PNG = 1 кадр, duration 1000 мс (`01 00 E8 03` — та самая «мета» из report.md §2.2!) | — |
| Доп. канал | `67 02` спектр 16 Гц | — | — |

Детали конвертера (`ConvertImageToBinary_RGB_Type_1.cs`):
- Заголовок STANDARD: `frames u16 LE`, затем `duration u16 LE` (мс) на кадр; длительность из
  PropertyTagFrameDelay GIF ×10; нулевая длительность заменяется на 62. Статика: `01 00 E8 03`.
- Статичная картинка (оба формата) поворачивается `Rotate90FlipNone` **перед** конверсией и
  **сохраняется на месте** (MusicInfo.png на диске = уже повёрнутый буфер).
- RGB565 LE: `r=(v>>11)&31<<3, g=(v>>5)&63<<2, b=v&31<<3`; при `screen_type==1` — перестановка
  битовых групп (для монохромной панели не важно).
- Итерация строк в рендере идёт в обратном порядке, но офсет считается от номера строки —
  раскладка буфера от этого не меняется (top-down).

### Что это значит для стороннего хоста (macOS)

1. **Произвольный полноэкранный контент с плавной анимацией = канал 0x61**: рендерим кадры,
   собираем мультикадровый бинарник (заголовок + кадры), льём один раз — устройство анимирует
   автономно, хост свободен. Скорость аплоада ~1 мс/чанк → полный экран (616 чанков) ≈ 0.6–0.7 с;
   дальше 0 FPS трафика.
2. **Быстро меняющийся контент** (часы, метрики, «видео» ~3–4 Гц) = `67 01` регион 48×N
   (218 чанков ≈ 230 мс на кадр 48×136) или `62 01` 85×N; плюс `67 02` бары 16 Гц.
3. Реализация на macOS не требует GearLink вообще: весь протокол — 64-байтные репорты в vendor-iface 1
   (report ID 0). KBAgent-конвейер (BASS/SMTC/рендер) воспроизводим нативно или не нужен вовсе.

### Загадка 0x6A/0x6B из захвата 2026-10-02

В .NET-компонентах (kbapi/kbagent/kbservice) опкоды 106/107 **не встречаются** — вся карта
`array[1] = N` собрана: {18, 34, 36, 37, 80, 81, 83, 84, 97, 98, 99, 100, 102, 103, 113, 117}.
`6A/6B` из старого захвата шли не из .NET-части GearLink (нативная SwAgentDll / легаси-путь).
Загрузка картинки из старого захвата целиком описывается CustomAnimation_RGB_Type_1
(`61 01` total=616 u32 LE → `61 02` чанки → `61 03`), а «6B BEGIN» был лишним/параллельным
правлением режима. Для стороннего хоста оба не нужны.

Смена режимов в GearLink: `LaunchOledFunction_Type_1` / `DisableOledRunningFunction_Type_1`
читают профильные `.txt` и запускают/останавливают `SetOledHardwareinfo_Type_2` (0x66),
`SetOledWinNotification`, `MusicMode_RGB_Type_1` — при каждом переключении профиля OLED-функции
перезапускаются (отсюда конфиг-пачки `0x24/0x50/0x63` из report.md §2.1: `0x24 01` = GetOledMode,
`50 40/55/60/61` = коммиты SetProfile, `0x63` = время).

## 8. Значение для стороннего хоста (произвольный контент на OLED)

| Канал | Скорость | Произвольность | Заметки |
|---|---|---|---|
| `67 02` спектр | **15.8 Гц** | нет (32–60 колонок-байтов) | готовый «живой» элемент, рисуется устройством |
| `67 00`+`67 01` инфо | **~3 fps** (полный кадр 177×48 = 297 мс; 136×48 ≈ 260 мс) | **полные пиксели** RGB565 | область 48(px высота)×(до 680) — это верх/низ экрана в mode 2/3 |
| `0x6B/0x61` full-frame (report.md) | ~616 чанков, медленнее | весь экран 184×97 | для статичных картинок |

Выводы:
1. **Произвольная графика с обновлением ~3 Гц** — реалистично через music-info канал уже сейчас:
   `67 00 (total,w,h)` + стрим `67 01` + (опционально) `67 03`.
2. Комбинированный режим 2/3 даёт **картинку сверху + живой 16-Гц спектр снизу** одновременно.
3. Широкий баннер (до 680×48) устройство **само прокручивает** — «бегущая строка» бесплатно.
4. RGB565 байтовый порядок для всех растров — **LE** (резолвит открытый вопрос §4 из `report.md`).

### Открытые вопросы (активные эксперименты следующего этапа)

- Можно ли слать `67 01` **без** повторного `67 00` (сбрасывает ли `67 00` указатель записи)?
- Примет ли прошивка `w>680`, другие `h`; что будет с 60 колонками в `67 02`?
- Скорость аплоада из стороннего процесса при живом GearLink (конфликт писателей HID) —
  вероятно, надо останавливать музыку в GearLink (on_off=0 через его же сокет 8300!) либо глушить задачи.
- KBAgent можно использовать как сервис: слать JSON на 127.0.0.1:8300 (class_id 10030000) и пусть сам
  рендерит/льёт — но тогда контент ограничен текстом. Для произвольных кадров — свой HID-писатель.
- Сон OLED / wake в music mode.

## 9. Файлы

| Файл | Что это |
|---|---|
| `13-music-mode.pcap` | 126 с: спектр 15.8 Гц, пауза→затухание→тишина, сменa трека: `67 00`+284 чанка за 297 мс |
| `13-music-mode-scenario.log` | тайминги нажатий play/pause/next (2-я серия нажатий действовала) |
| `14-music-trackchange.pcap` | 2 нажатия next-track без эффекта (плеер не обрабатывает NEXT) |
| `15-music-scroll.pcap` | 25 с прокрутки длинного названия: 0 перезаливок → скролл на устройстве |
| `analysis/reassemble_capture.py` | сборка буфера из чанков дампа (учёт обратного порядка) |
| `analysis/decode_music_bitmap.py` | декод RGB565 → PNG (LE/BE) |
| `decode565.ps1`, `rot.ps1` | декод RGB565 → PNG (System.Drawing) + поворот 90° |
| `smtc.ps1` | чтение SMTC-сеанса (PS 5.1, AsTask-хелпер) |

## 10. Воспроизведение анализа

```bash
T="/c/Program Files/Wireshark/tshark.exe"
# команды хост→клавиатура
"$T" -r 13-music-mode.pcap --disable-protocol usbhid \
  -Y "usb.device_address==2 && usb.endpoint_address==0x01 && usb.capdata" \
  -T fields -e frame.time_relative -e usb.capdata > out.txt
# ACK-канал
"$T" -r 13-music-mode.pcap --disable-protocol usbhid \
  -Y "usb.device_address==2 && usb.endpoint_address==0x82 && usb.capdata" \
  -T fields -e frame.time_relative -e usb.capdata > in.txt
# всплеск аплоада (подкоманды 67 00/67 01)
awk -F'\t' '$2 ~ /^670[01]/' out.txt > burst.txt
python analysis/reassemble_capture.py burst.txt frame.raw   # 48 × (w) портрет
powershell -File decode565.ps1 -in frame.raw -out f.png -W 48 -H 177
powershell -File rot.ps1 -in f.png -out f_rot.png           # 90° CW → читаемый текст
```
