# Живой MiniDV на macOS 26 (Apple Silicon)

**Sony DCR-PC115E по FireWire/i.LINK: видео и звук в реальном времени — в OBS, Chrome, Zoom.**
Плюс методика побитовой оцифровки лент на машине, из которой Apple выпилила весь стек FireWire.

[English version →](README.md) · [Подробный гайд (HTML)](https://paperpit.github.io/minidv-macos/guide.ru.html) · [Настройка живого режима](docs/live-setup.md)

---

## Зачем это

Apple удалила поддержку FireWire из macOS 26 (Tahoe). Старые MiniDV-камкордеры остались без драйвера:
ни оцифровать ленту, ни использовать камеру как веб-камеру. Здесь собрано то, что получилось восстановить
на MacBook Air M1 — с проверкой результата числами, а не «вроде работает»:

- **оцифровка ленты** — побитовая копия DV, файл делится на 144 000 байт без остатка;
- **живое видео** — камкордер в режиме CAMERA или на воспроизведении идёт в OBS через Syphon;
- **живой звук** — прямо из DV-потока, без отдельного аудиокабеля, в BlackHole (приложения видят его
  как микрофон) — и дальше в OBS, Chrome, Zoom;
- **один клик** — ярлык поднимает камеру, звук, источники в OBS и виртуальную камеру для браузера.

Проверено побитовой сверкой с FFmpeg: звук 16 бит/48 кГц и 12 бит/32 кГц, офлайн-рендер графа вывода.
Геометрия кадра — отдельная история: `16:9WIDE` у Sony пишется анаморфно, и без растяжки картинка
получается сжатой по ширине на 42 % (подробно в гайде).

## Что работает

| Возможность | Состояние |
|---|---|
| Драйвер FireWire (DriverKit, user-space OHCI) | работает: `net.mrmidi.ASFW.ASFWDriver [activated enabled]` |
| Захват DV с шины | работает: кольцо 480-байтных DIF-чанков в разделяемой памяти, канал 63 |
| Оцифровка ленты в `.dv` | работает: 1895 кадров, 1 потерянный, 0 переполнений, остаток от деления на 144 000 = 0 |
| Живое видео в OBS | работает через Syphon, 50p после деинтерлейса, квадратный пиксель |
| Живой звук из DV | работает: 16 бит/48 кГц и 12 бит/32 кГц, побитово как у FFmpeg |
| Виртуальная камера для браузера | работает: OBS Virtual Camera (+ автостарт ярлыком) |
| Камера как системное устройство (CMIO) | **нет**: расширение написано, но не активируется ad-hoc-подписью |
| Вторая звуковая пара (ST2), общий таймлайн, bus reset | **нет**, см. «Что осталось» |

## Что нужно

**Железо**

- Mac на Apple Silicon (проверено на MacBook Air M1, `MacBookAir10,1`).
- Камкордер Sony DCR-PC115E (или другой MiniDV с i.LINK 4-pin, S100).
- Кабель FireWire 4-pin ↔ 9-pin.
- **Apple Thunderbolt → FireWire** (A1463) и **Apple Thunderbolt 3 → Thunderbolt 2** (A1790).
  Порядок сборки жёсткий: `A1790 в Mac → A1463 в него → кабель в камеру`. A1790 однонаправленный.
- Питание камеры от сети: просадка на батарее в середине кассеты = обрыв захвата.

**Софт**

```bash
xcode-select --install                        # Xcode и xcodegen
brew install xcodegen
brew install --cask blackhole-2ch             # виртуальный микрофон для звука с камеры
brew install --cask obs                       # OBS Studio 30+ (виртуальная камера — системное расширение)
brew install node                             # для ярлыка (obs-websocket); есть в системе — не нужно
```

`node` нужен только помощнику ярлыка: без него ярлык всё равно поднимет камеру и звук, но источники
в OBS придётся добавить руками (один раз, OBS их запомнит).

## Быстрый старт

### 1. Драйвер

```bash
git clone https://github.com/mrmidi/ASFireWire.git ~/Developer/ASFireWire
cd ~/Developer/ASFireWire
git checkout 9055449                          # коммит, на котором снят наш патч
git apply ../minidv-macos/patches/asfw-minidv-live.patch
./build.sh --config Release && ./sign.sh
```

Дальше по гайду: отключить SIP, включить developer mode, установить `ASFW.app`, одобрить расширение
в «Системных настройках» и перезагрузиться. Пошагово — [docs/guide.ru.html](https://paperpit.github.io/minidv-macos/guide.ru.html),
разделы 2 и 6.

### 2. Живой режим

```bash
git clone https://github.com/PaperPit/minidv-macos.git && cd minidv-macos
./tools/minidv-live/install.sh                # соберёт и поставит «MiniDV Live.app»
```

Двойной клик по **MiniDV Live** (или ⌘Space → «MiniDV Live») — и всё поднято: камера, звук,
источники в OBS, виртуальная камера. Лог: `~/Library/Logs/MiniDVLive.log`.

В OBS появятся:

- `Sony DCR-PC115E` (Syphon Client) — видео;
- `BlackHole 2ch` (Audio Input Capture) — звук.

В Chrome/Zoom выбирайте камеру `OBS Virtual Camera`, микрофон `BlackHole 2ch`.

> **Ловушка.** `OBS Virtual Camera` — системное расширение, оно **всегда** видно в списке камер,
> даже когда OBS закрыт. Кадры идут только при активном выходе виртуальной камеры: сначала
> **Start Virtual Camera** в OBS, потом выбирать в браузере. Ярлык делает это сам.
> Расширение объявляет единственный формат 1920×1080@60 — если приложение требует ровно 30 к/с,
> будет отказ (`tools/camera-test` покажет это точно).

## Проверки (без камеры)

```bash
ASFW=/Applications/ASFW.app/Contents/MacOS/ASFW

$ASFW --list-audio-devices                     # куда играть звук
$ASFW --audio-selftest 3                       # тон 1 кГц слева / 3 кГц справа в приёмник
$ASFW --dv-audio-selftest test48.dv --pcm-out out.pcm        # .dv → звук, реальный путь
$ASFW --audio-render-selftest test48.dv --pcm-out render.pcm # тот же граф офлайн
$ASFW --dv-frame-selftest wide.dv --png wide.png             # геометрия кадра: 1024×576 для 16:9
```

Тестовые потоки генерируются FFmpeg:

```bash
# 16 бит/48 кГц, 4:3
ffmpeg -y -f lavfi -i "testsrc2=size=720x576:rate=25" \
  -f lavfi -i "sine=frequency=1000:sample_rate=48000:duration=10.5" \
  -f lavfi -i "sine=frequency=3000:sample_rate=48000:duration=10.5" \
  -filter_complex "[1:a][2:a]join=inputs=2:channel_layout=stereo[a]" \
  -map 0:v -map "[a]" -c:v dvvideo -pix_fmt yuv420p -r 25 \
  -c:a pcm_s16le -ar 48000 -ac 2 -t 10 -f dv test48.dv

# то же, но анаморфный 16:9
ffmpeg -y -f lavfi -i "testsrc2=size=720x576:rate=25" -c:v dvvideo -pix_fmt yuv420p -r 25 -aspect 16:9 -t 2 -f dv wide.dv
```

## Что внутри

```
README.md / README.ru.md   что это и как поставить
docs/guide.ru.html         подробный гайд: стек, драйвер, баги, методика оцифровки, живой режим
docs/live-setup.md         настройка живого режима, флаги, диагностика
docs/troubleshooting.md    типичные проблемы и что смотреть
tools/minidv-live/         ярлык «MiniDV Live»: запуск камеры, звука, OBS и виртуальной камеры
tools/camera-test/         страница проверки камеры и микрофона в браузере
patches/                   патч к ASFireWire: живой DV, звук, геометрия, камерное расширение
images/                    картинки для документации
```

## Что осталось

- **Камера как системное устройство (CMIO).** Расширение `ASFWCamera` публикует «Sony DCR-PC115E»,
  но не активируется: `OSSystemExtensionError.validationFailed (9)` — нужна нормальная подпись
  (Developer ID/Development и совпадающий Mach-service), а не ad-hoc. Поэтому живой режим идёт
  через Syphon → OBS → OBS Virtual Camera; работает везде, включая браузер.
- **ST2** (вторая звуковая пара 12-битного режима) не публикуется; разбирается первая.
- **Общий таймлайн** звука и видео не вводился: синхронность держится на покадровом приходе,
  метрики дрейфа нет.
- **Bus reset**: watchdog есть у Syphon-раннера, обработки выдёргивания кабеля и CAMERA↔VCR нет.
- **gap count 44** при оптимальном 5 — драйвер не претендует на роль Bus Manager.
- **NTSC и DVCPRO50** не поддержаны в извлечении звука (только PAL 625/50).

## Безопасность и откат

Живой режим требует dext, подписанного ad-hoc, а значит **выключенного SIP** и включённого
developer mode. Это рабочее состояние для задачи, но не для повседневной машины: не пользуйтесь
в таком виде банковскими приложениями. Порядок полного отката — в гайде, раздел 6; коротко:
сначала снять расширение, потом `csrutil enable`, иначе возможен осиротевший kernel-stub.

## Лицензия и благодарности

Apache License 2.0 — см. [LICENSE](LICENSE) и [NOTICE](NOTICE).

Проект стоит на плечах [ASFireWire](https://github.com/mrmidi/ASFireWire) (DriverKit-драйвер FireWire
для macOS 26) — без него ничего бы не было. Также использованы
[Syphon](https://syphon.github.io/) (передача кадров в OBS),
[BlackHole](https://existential.audio/blackhole/) (виртуальный микрофон),
[obs-websocket](https://github.com/obsproject/obs-websocket) (настройка сцены),
[FFmpeg](https://ffmpeg.org/) (эталон для сверки звука и геометрии, генерация тестовых потоков).
