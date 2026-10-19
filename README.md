# Nothing Matrix Music Visualizer

Music visualiser for the Nothing Phone 3 Glyph Matrix (25×25). Draws a live
audio spectrum on the back LED matrix, monochrome — brightness varies, colour
does not. Audio comes from the system output via `Visualizer(0)`, so there is
no screen recording, no microphone, and nothing leaves the device except the
optional lyrics lookup (off by default).

**Author:** Danil Evrikov

## Download

Version **0.1** — the first public release. The signed APK is attached to
[Releases](../../releases). Requires Android 14+ (minSdk 34), a Nothing Phone 3
and NothingOS 20250801 or newer.

If a build signed with a different key is already installed, uninstall it first:
Android will not replace an app across signatures.

## What it does

- 22 visualisation modes (`BARS`, `PEAKS`, `FLOW`, `RAIN`, `MIRROR`, `ECHO`,
  `TILES`, `FRAME`, `ARC`, `RADIAL`, `RINGS`, `HALO`, `VU`, `STAR`, `ZIGZAG`,
  `SCOPE`, `LISSAJOUS`, `XSHAPE`, `WALK`, `WAVE`, `PULSE`, `TEXT`), each with
  its own description shown under the preview.
- Per-band sensitivity, attack/release, brightness, contrast, auto-brightness,
  pulse depth, frame rate (20/30/60/120 FPS target) — all live-tunable.
- `TEXT` mode: song title and artist scroll across the matrix; with "Song
  lyrics" enabled, lyrics appear **one word at a time, in time with the
  singer**, centred. A word that fits stays on one line, a longer one wraps onto
  two lines with a hyphen, and a word too long even for that is shown syllable
  by syllable. A word stays lit for exactly the span it is sung: line ends are
  taken from the LRC itself (including its empty timestamps), and timing inside
  a line is counted in syllables, not letters. A manual offset (–5…+5 s)
  corrects per-track drift.
- `TEXT` never leaves the matrix black between lines: pick any other visualiser
  as the filler and it runs the intro, the solos and the tail of the song until
  the singing starts again.
- Measured FPS shown next to requested FPS in the header; a red line flags
  when you're asking for more than the Nothing system service can deliver.
- Diagnostics tab to probe system-audio capture without clashing with the
  visualiser.

## Privacy

- Audio is **never recorded or sent anywhere**. The app hears exactly what the
  phone outputs — nothing more.
- No analytics.
- The **only** network request is the optional lyrics lookup via [LRCLIB]
  (free, no key, no account), and only when you explicitly turn "Song lyrics"
  on. It sends the track title and artist; nothing else.

## Requirements

- Nothing Phone 3 with **NothingOS 20250801 or newer**. Older versions silently
  ignore matrix control.
- The app must be launched at least once after a reboot — there is no
  auto-start.
- For AOD and Glyph Button support, open **Settings → Glyph Interface** and
  move **Nothing Matrix Music Visualizer** from *Disabled* to *Active*.

## Build

Gradle 8.11.1, AGP 8.10.1, Kotlin 2.0.0, compile/targetSdk 35, minSdk 34.

The JDK is **not** on PATH — set it before every Gradle call:

```powershell
$env:JAVA_HOME="C:\Users\talmo\.jdks\jbr-21.0.11"
$env:PATH="$env:JAVA_HOME\bin;$env:PATH"
cd C:\dev\matrix_test\exemple
.\gradlew.bat :app:testDebugUnitTest   # unit tests (render, LRC, clock)
.\gradlew.bat assembleDebug           # debug APK (~27 MB)
.\gradlew.bat installDebug            # install on connected phone

# signed release (needs keystore.properties — see RELEASING.md)
.\gradlew.bat :app:assembleRelease    # signed APK
.\gradlew.bat :app:bundleRelease      # AAB
```

Open **`exemple/`** in Android Studio — never `exemple/app/`, or Gradle sync
breaks.

### Render check without a phone

`Spectrum.kt` has no Android dependencies, so its rendering is verified by a
unit test that writes ASCII previews of every mode:

```powershell
.\gradlew.bat :app:testDebugUnitTest
Get-Content app\build\mode-previews.txt
```

## Lyrics

Lyrics come from [LRCLIB] (`lrclib.net`) — free, no account, no key. The lookup
is opt-in and defaults to **off**; the app never touches the network otherwise.

Matching a found recording to the playing one is done **by duration**, not by
name — studio, live, remix and bootleg all share a title. A manual offset
(–5…+5 s) corrects per-track timing drift.

## Design notes

- Always-dark theme: the visualiser reads as a light source, the UI carries
  the colour.
- Matrix is physically a circle inside the 25×25 square — dead corners are
  masked at render time.
- Two-rate audio pipeline: `pushFft()` at ~20 Hz measurement, `advance()` per
  rendered frame for smoothing.

[LRCLIB]: https://lrclib.net

---

## Русский

Музыкальный визуализатор для Nothing Phone 3 Glyph Matrix (25×25). Рисует
живой аудиоспектр на светодиодной матрице сзади телефона. Одноцветный: меняется
только яркость, цвет не меняется. Звук берётся из системного выхода через
`Visualizer(0)` — нет записи экрана, нет микрофона, ничего не покидает
устройство, кроме опционального поиска слов песни (выключен по умолчанию).

**Автор:** Данил Евриков (Danil Evrikov)

### Скачать

Версия **0.1** — первый публичный релиз. Подписанный APK прикреплён к
[Releases](../../releases). Нужны Android 14+ (minSdk 34), Nothing Phone 3 и
NothingOS 20250801 или новее.

Если уже установлена сборка с другим ключом, сначала удалите её: Android не
заменяет приложение с другой подписью.

### Что делает

- 22 режима визуализации (`BARS`, `PEAKS`, `FLOW`, `RAIN`, `MIRROR`, `ECHO`,
  `TILES`, `FRAME`, `ARC`, `RADIAL`, `RINGS`, `HALO`, `VU`, `STAR`, `ZIGZAG`,
  `SCOPE`, `LISSAJOUS`, `XSHAPE`, `WALK`, `WAVE`, `PULSE`, `TEXT`), у каждого
  своё описание под превью.
- Чувствительность, attaque/release, яркость, контраст, авто-яркость, глубина
  пульсации, частота кадров (20/30/60/120) — на лету.
- Режим `TEXT`: название и исполнитель бегут по матрице; с «Слова песни» —
  слова по одной штуке, в такт с певцом, по центру. Слово, которое влезает,
  остаётся на одной строке, длинное переносится на две строки с дефисом, а
  слишком длинное для двух строк показывается по слогам. Слово горит ровно
  столько, сколько поётся: конец строки берётся из самого LRC (включая пустые
  отметки), а время внутри строки считается по слогам, а не по буквам. Ручной
  сдвиг (–5…+5 с) корректирует задержку на трек.
- `TEXT` не оставляет матрицу чёрной между строками: выберите любой другой
  визуализатор — он играет вступление, проигрыши и конец песни, пока певец
  снова не запоёт.
- Измеренный FPS рядом с запрошенным в шапке; красная строка предупреждает,
  если просите больше, чем может Nothing-сервис.
- Вкладка «Диагностика» для проверки захвата звука без конфликта с визуализатором.

### Приватность

- Аудио **никогда не записывается и не отправляется никуда**. Приложение слышит
  ровно то, что телефон выводит — ничего больше.
- Нет аналитики.
- Единственный сетевой запрос — опциональный поиск слов через
  [LRCLIB] (бесплатно, без ключа и аккаунта), и только если вы включили
  «Слова песни». Передаются название и исполнитель трека; ничего другого.

### Требования

- Nothing Phone 3 с NothingOS **20250801 или новее**. На старых версиях
  управление матрицей молча игнорируется.
- После перезагрузки приложение нужно запустить хотя бы раз — автозапуска нет.
- Для AOD и кнопки Glyph: **Настройки → Glyph Interface**, перетащите
  «Nothing Matrix Music Visualizer» из *Disabled* в *Active*.

### Сборка

Gradle 8.11.1, AGP 8.10.1, Kotlin 2.0.0, compile/targetSdk 35, minSdk 34.

JDK не в PATH — установите перед каждым вызовом Gradle:

```powershell
$env:JAVA_HOME="C:\Users\talmo\.jdks\jbr-21.0.11"
$env:PATH="$env:JAVA_HOME\bin;$env:PATH"
cd C:\dev\matrix_test\exemple
.\gradlew.bat :app:testDebugUnitTest   # юнит-тесты (рендер, LRC, часы)
.\gradlew.bat assembleDebug           # debug APK (~27 МБ)
.\gradlew.bat installDebug            # на подключённый телефон

# подписанный релиз (нужен keystore.properties — см. RELEASING.md)
.\gradlew.bat :app:assembleRelease    # подписанный APK
.\gradlew.bat :app:bundleRelease      # AAB
```

Открывайте папку **`exemple/`** в Android Studio — не `exemple/app/`, иначе
сборка сломается.

### Проверка рендера без телефона

`Spectrum.kt` не зависит от Android, поэтому рендер проверяется юнит-тестом,
который пишет ASCII-превью всех режимов:

```powershell
.\gradlew.bat :app:testDebugUnitTest
Get-Content app\build\mode-previews.txt
```

### Слова песни

Слова берутся из [LRCLIB] (`lrclib.net`) — бесплатно, без ключа и аккаунта.
Поиск опционален и **выключен по умолчанию**; без него приложение в сеть не
ходит. Совпадение записи с Playing треком подбирается **по длительности**, а не
по названию — студия, концерт, ремикс и бутлег носят одно имя. Ручной сдвиг
(–5…+5 с) исправляет задержку на трек.

### Заметки по дизайну

- Всегда тёмная тема: визуализатор читается как источник света, цвет — в UI.
- Матрица физически — круг внутри квадрата 25×25, мёртвые углы гасятся при
  рендере.
- Два темпа аудио: `pushFft()` на измерение ~20 Гц, `advance()` на каждый
  кадр для сглаживания.

[LRCLIB]: https://lrclib.net
