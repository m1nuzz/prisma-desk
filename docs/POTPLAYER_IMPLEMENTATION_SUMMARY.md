# PotPlayer implementation summary

## Выполненные изменения

| Область | Реализация |
|---|---|
| Нативный Windows слой | Добавлен `src-tauri/shared/services/player.rs`: проверка PotPlayer-пути, auto-detection через App Paths/типовые каталоги, безопасный запуск с раздельными аргументами, `/new`, `/seek`, чтение duration/position/status через `SendMessageTimeoutW`, выбор окна с PID fallback, per-message telemetry и `0x5005` seek. |
| Tauri API | В `core/lib.rs` добавлены команды `player_detect`, `player_choose_path`, `player_validate`, `player_start`, `player_read_state` и `player_seek`. |
| UI и bridge | `bridge.js` предоставляет `desktopAPI.player`; настройка Prisma позволяет выбрать PotPlayer автоматически, вручную, VLC или обычный внешний плеер. Выбор применяется к video, torrent и IPTV keyspace. |
| Resume/progress | `client-inject.js` создаёт одну внешнюю playback-сессию, опрашивает PotPlayer каждые 2 секунды, сохраняет прогресс с debounce, делает final flush и не записывает нулевой таймкод при timeout. |
| Prisma watched/history | При доступном `Prisma.Timeline` позиция записывается через `Prisma.Timeline.update({ hash, percent, time, duration })`. Это тот же публичный контракт, который Prisma использует для продолжения просмотра, таймлайна на карточках, истории и account sync. Локальное `prisma_desktop_external_progress_v1` остаётся fallback. Snapshot `window.__desktop_external_progress_diagnostics` фиксирует reads, fallback window, Timeline writes, final flush и resume position. |
| Документация | Добавлены аудит, повторный IPC re-audit, E2E test plan, инструкция пользователя, базовый smoke test и строгий `potplayer-ipc-diagnostic.ps1`. |

## Фактические проверки

| Проверка | Результат |
|---|---|
| JavaScript parser | `node --check src-tauri/module/client-inject.js` и `node --check src-tauri/module/bridge.js` прошли успешно. |
| Rust formatting | `cargo fmt --all -- --check` прошёл успешно. |
| Rust unit tests | `cargo test --manifest-path src-tauri/Cargo.toml player::tests`: 2/2 успешно. |
| Native host check | `cargo check --manifest-path src-tauri/Cargo.toml` успешно. |
| Windows cross-check | `cargo check --target x86_64-pc-windows-gnu` успешно, включая `winreg` и весь `#[cfg(windows)]` IPC код. |
| Native Windows PotPlayer IPC | Успешный [workflow 31680974020](https://github.com/m1nuzz/prisma-desk/actions/runs/31680974020) запустил предоставленный PotPlayer на `windows-latest`, получил шесть меняющихся timestamp, duration/status и успешный production `/seek` resume. |
| Prisma session manager | `node scripts/verify-external-progress-manager.js` подтвердил `Prisma.Timeline.update`, локальный fallback, watched threshold, resume policy и diagnostic snapshot в изолированном JS-тесте. |

## Строгий runtime gate

Не считать задачу завершённой только потому, что PotPlayer открылся. Native IPC и production `/seek` уже прошли на Windows runner с предоставленным PotPlayer; для конкретного Prisma-аккаунта остаётся запустить клиент, подтвердить `successfulReads >= 5`, `timelineUpdates >= 1` и resume round trip в `window.__desktop_external_progress_diagnostics`. Полный протокол находится в `docs/POTPLAYER_E2E_TEST_PLAN.md`, а native evidence — в `docs/POTPLAYER_NATIVE_WINDOWS_TEST_RESULT.md`.

## Что необходимо сделать на Windows 11 перед выпуском

Native `WM_USER` взаимодействие с предоставленным PotPlayer уже подтверждено на реальном Windows runner. На собственной Windows 11 выполните следующий сценарий, чтобы подтвердить именно вашу установку PotPlayer и live Prisma Timeline/account session:

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\scripts\potplayer-ipc-diagnostic.ps1 `
  -PotPlayerPath 'C:\Program Files\PotPlayer\PotPlayerMini64.exe' `
  -Url 'https://<ваш-короткий-тестовый-медиа-url>'
```

Успешный результат — не менее пяти показаний с возрастающим `positionSec`, разумным `durationSec` и `status=2` во время воспроизведения. После этого вручную проверьте путь Prisma: открыть фильм, посмотреть примерно 30 секунд, закрыть PotPlayer, открыть тот же контент и убедиться, что запуск идёт с сохранённой позиции; затем досмотреть почти до конца и убедиться, что Prisma показывает просмотренный статус.

> Не запускайте один процесс «от имени администратора», а другой — без повышения прав. Windows может заблокировать межпроцессные сообщения между разными уровнями UAC.
