# AGENTS.md

Android music visualizer for Glyph Matrix (Nothing Phone 3, 25×25).
Root: `C:\dev\matrix_test`; the Gradle project lives in `exemple/`.
No git locally; repo prepared for GitHub at `C:\dev\matrix_test/`.
Author: Danil Evrikov. Single `:app` module. `exemple` is a typo in the original — do not rename it.

## Release files (for GitHub)

- `.gitignore` — Gradle, Android Studio, keystores, `keystore.properties`, `dist/`, secrets
- `LICENSE` — MIT, copyright Danil Evrikov
- `README.md` — English + Russian, download, build instructions, privacy, design notes
- `RELEASING.md` — keystore, GitHub secrets, tagging; the APK is signed, keep `release.keystore` backed up
- `.github/workflows/ci.yml` — GitHub Actions: JDK 21, unit tests, debug APK build, artifact upload
- `.github/workflows/release.yml` — on `v*` tag: tests, signed release APK, GitHub release upload (needs the 4 secrets from `RELEASING.md`)
- `exemple/keystore.properties.example` — signing template; the real file and `*.keystore` are gitignored
- `exemple/app/src/main/res/drawable/ic_launcher_*.xml` — new adaptive icon: black background, red Nothing-style corner dots, 5×5 dot-grid foreground

Version **0.1** (`versionCode = 1`) — first public release.


## Build & test

Open `exemple/` in Android Studio (not `exemple/app/` — sync breaks otherwise).

JDK is **not** on PATH. Set it before every Gradle call:

```powershell
$env:JAVA_HOME="C:\Users\talmo\.jdks\jbr-21.0.11"
$env:PATH="$env:JAVA_HOME\bin;$env:PATH"
cd exemple
```

- `.\gradlew.bat :app:testDebugUnitTest` — all unit tests (render, LRC, clock)
- `.\gradlew.bat :app:testDebugUnitTest --tests "..."` — one test
- `.\gradlew.bat assembleDebug` — APK
- `.\gradlew.bat installDebug` — to phone

Gradle marks tests UP-TO-DATE when nothing changed; add `--rerun` if you edited `Spectrum.kt` and expect a fresh `app/build/mode-previews.txt`.
`Spectrum.kt` has no Android deps — it is the only part verifiable without a phone.

Environment gotchas:
- Android SDK at `%LOCALAPPDATA%\Android\Sdk` (platform 35, build-tools 35); `adb` in `platform-tools`.
- `exemple\local.properties` holds an absolute `sdk.dir` — moving the project doesn't break it.
- Gradle JDK must be Embedded 21. Gradle 8.11.1 / AGP 8.10.1 do not work with system JDK 25 or wrapper 9.x.
- `android.overridePathCheck=true` in `gradle.properties` (AGP rejects Cyrillic paths).
- **Do not move the project back into OneDrive** — Gradle cannot delete `build/` there.

## AARs

`exemple/app/libs/` has two AARs. Only `glyph-matrix-sdk-1.0.aar` is wired up; `glyphsdk_0606.aar` lacks `setAppMatrixFrame()` and is unused. NothingOS 20250801+ required, otherwise `setAppMatrixFrame()` silently no-ops.

## Audio

Sound comes from `Visualizer(0)` (global output mix) — no MediaProjection, no microphone.
`RECORD_AUDIO` + `MODIFY_AUDIO_SETTINGS` permissions required.
`getMaxCaptureRate()` = 20000 (mHz), `captureSize` = 1024, ~23 callbacks/s. `sampleRate` reports `48000000` — do not use it for band math.
**Never run `VisualizerService` and `AudioProbe` at the same time** — they fight over the one output mix.

## Matrix & render

Matrix is 25×25 = 625 ints (`row * 25 + col`), physically a circle inscribed in the square — corners are dead pixels.
Normalize distances on `MATRIX_R = MID`, not corner distance; `render()` ends with `maskToCircle()`.
Color encoding in `Spectrum.pack()`: `R<<10 | G<<5 | B`, 5 bits/channel (max 31). UNVERIFIED on hardware — check `Spectrum.calibrationFrame()`; `pack()`/`CH` are the only places to change if wrong.

Pipeline: FFT → `pushFft()` → band targets → `advance()` (each frame) → 25 bands → `render()` → `setAppMatrixFrame(int[])`.
`pushFft()` tops out ~20 Hz; `advance()` interpolates per frame. Two rates on purpose.

## Adding a VizMode

`Spectrum.kt` must stay pure Kotlin (unit-tested). Names/descriptions live in resources, not the enum.
1. Add enum constant to `VizMode` (22 now).
2. Add branch in `Spectrum.render()`.
3. Add matching `when` branch in `SpectrumRenderTest` (no `else` — compiler catches omissions).
4. Add `mode_<NAME>` / `mode_<NAME>_desc` in `values/strings.xml` and `values-en/strings.xml`.
5. Add two lines in `modeTitle`/`modeDesc` in `MainActivity.kt`.

`Spectrum` is a singleton shared by all tests; `reset()` does **not** clear the TEXT band. Every test must set the band itself.

## TEXT mode timing (the part that keeps getting "fixed")

Goal: a word is on the matrix for exactly the span it is sung. LRC only gives line starts, so:

1. **Line ends** come from the *next timestamp in the file*, marker or not. Timed blank lines (`[00:33.00]` with no words) are kept as `Line.isMarker` — they are the file saying "the singing stopped here". Dropping them is what stretched a line across the following instrumental. `typicalSlot()` skips gaps touching a marker; `spokenEndOf()` obeys them, but steps over a marker less than `MIN_MARKER_MS` after its line.
2. **Inside a line**, `WordCues` weights by `Syllables.count()` (vowel *runs*), not letters. A line is one cue per word — or per syllable piece for a word past `SYLLABLE_LIMIT`; the cues are contiguous, with no blank between them, so nothing is blanked early.
3. **Long words** go through three levels. `TextStrip.rasterizeWord()` draws a word that fits on one line; a word that would need to shrink below `WRAP_BELOW_SCALE` is wrapped across two lines with a hyphen (`hyphenBreak()`, balanced halves); and a word longer than `Syllables.SYLLABLE_LIMIT` (6 letters) is cut into at most `Syllables.MAX_PIECES` (4) syllable pieces by `Syllables.forLineGrouped()` and shown one at a time. Each piece is then drawn by the same one-line/two-line rule. The two-line strip fills rows `TWO_LINE_TOP..TWO_LINE_BOTTOM`, so `Spectrum.renderText()` scans all 25 rows.
4. `PlaybackClock` must never produce garbage: `lastPositionUpdateTime` of 0/wall-clock is ignored, `playbackSpeed` of 0 while playing means 1×, and a paused seek resyncs (`resyncMs`).

`textSpeed` only scrolls the *title* (no lyrics); it does not affect word timing. `lyricsOffsetMs` (slider «Сдвиг слов», ±5 s, plus = earlier) is the only per-song correction and is applied in `VisualizerService.lyricPosition()` alone.

## Service lifecycle

`MainActivity` = UI + permissions only; all work is in `VisualizerService` (foreground, `specialUse`).
Manifest requires `com.nothing.ketchum.permission.ENABLE`, `<meta-data android:name="NothingKey">`, and `<property android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE">`.
Service also registers as Glyph Toy (`com.nothing.glyph.TOY`, `aod_support=1`); `onBind` must return a non-null binder or Glyph Button events die. After install, move the toy from Disabled to Active in Settings → Glyph Interface.
On stop: `closeAppMatrix()` → `turnOff()` → `unInit()`. An empty frame does **not** release the matrix — call `closeAppMatrix()` explicitly, else AOD toy won't appear and the user sees a black square.
No auto-start after reboot.

## Settings

`VizSettings` — 13 fields in `SharedPreferences("visualizer")`; 8 reach `Spectrum` via `configure()`.
`language` uses `LocaleManager.applicationLocales` (not AppCompat); the setter recreates the Activity — save language **before** applying it. `lyricsEnabled` defaults off (it is the only network call).
`idleMode` (any mode except `TEXT`, default `PULSE`) is what `frameMode()` draws when `TEXT` has no word on the matrix — intro, instrumental, tail. It is applied in `VisualizerService.pushFrame()` per frame, not to `Spectrum` state, so it needs no rebuild when the line changes.
Sliders commit on `onValueChangeFinished`, not `onValueChange` — a `remember`d draft inside `setContent` makes them snap back.

## UI threads

Service state is split into flows: `status` (rare), `bands` (every frame), `fpsFlow`/`bpmFlow` (every frame, FPS line only). Do not merge them into one root scope — the whole screen would recompose every frame.
Preview is one `Canvas`, not 625 `Box` composables.

## Build traps already fixed (don't reopen)

- `settings.gradle.kts` has `FAIL_ON_PROJECT_REPOS` — a module-level `repositories {}` kills the build.
- SDK auto-download can hang silently (0-byte zip in `.temp`); install via SDK Manager instead.
- Do not pipe `.kt` sources through `Get-Content | Set-Content` in PowerShell — it rewrites in system encoding and corrupts Cyrillic without compile errors. Edit in place; keep data files UTF-8.

## Removed modes (do not resurrect)

`VINYL`, `MASK`, `SHIMMER` were deleted by user decision and removed from `Spectrum.kt` entirely.
