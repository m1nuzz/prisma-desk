# Аудит Prisma Desk: внешние плееры и прогресс

## Подтверждённый стек

Проект — **Tauri 2 + Rust + JavaScript**, а не Electron. Frontend Prisma загружается во внешний WebView, после чего Tauri инжектирует `bridge.js` и `client-inject.js`. Windows-сборка должна использовать Rust-слой `src-tauri/core/lib.rs`; в текущем `Cargo.toml` нет зависимости для Windows Registry или Win32 window messaging.

## Текущий поток запуска

1. В `src-tauri/core/lib.rs` команда `find_player` обнаруживает только VLC по фиксированным путям `ProgramFiles`/`ProgramFiles(x86)`.
2. `bridge.js` экспортирует `window.desktopAPI.findPlayer()`, которая записывает найденный путь в Prisma `localStorage` и `Prisma.Storage` как `player_nw_path`, а тип игрока принудительно ставит в `player_torrent = "other"`.
3. `client-inject.js` патчит `Prisma.Player.play` и `Prisma.Player.iptv`.
4. При `player === "other"` либо Windows VLC с сохранённым путем вызывается `window.require("child_process").spawn(playerPath, [encodeURI(safeUrl)])`.
5. `bridge.js` эмулирует `child_process.spawn` через Tauri-команду `child_process_spawn`, а Rust запускает процесс с раздельными аргументами. Это уже безопаснее shell-конкатенации, но общий spawn не возвращает JS handle с PID/состоянием, достаточный для наблюдения за PotPlayer.
6. Для `vlc`, `mpv`, `iina`, `infuse` и других известных URI используется protocol opener; собственного observer таймкода во внешнем плеере в проекте нет.

## Точки настроек

В `client-inject.js` есть кнопка «Поиск плеера», фактически описанная как «Автоматически найти VLC». Она добавлена в Prisma Settings и вызывает `desktopAPI.findPlayer()`. Отдельного выбора player id/path/sync mode нет. Persistent Rust store (`store.json`) содержит только Prisma URL, fullscreen, auto-update, window state и TorrServer keys; typed external-player/session keys отсутствуют. Prisma player settings живут в Prisma `localStorage`, поэтому новая конфигурация должна учитывать оба слоя и импорт/экспорт настроек.

## Прогресс и watched

В переданных исходниках отсутствуют вызовы `Prisma.Player`/bridge для чтения внешнего таймкода, polling, сохранения позиции или `markWatched` в рамках внешнего запуска. Поэтому текущая PotPlayer-проблема не является одной неверной командой запуска: Prisma Desk сейчас запускает внешний процесс и сразу теряет контроль над ним. Для правильного решения нужен Tauri/Rust Windows IPC слой и JS session/progress manager, который свяжет `mediaId`/`episodeId` с конкретной сессией.

## Обязательная архитектура изменений

Нужно добавить:

- Rust Windows-only player service для обнаружения PotPlayer, проверки пути, запуска с отдельным массивом аргументов и чтения текущего состояния через Win32 messages с timeout.
- Tauri-команды и bridge API для `detect/validate/start/read_state/stop_observation`.
- Конфигурацию с явным `player id`, `path`, `mode` и `syncEnabled`, не выводя тип адаптера только по имени exe.
- Общий JS session manager с опросом примерно каждые 2 секунды, debounce записи прогресса и финальным flush при закрытии/исчезновении процесса.
- Использование существующего Prisma progress/watched API, если оно реально предоставляется загруженной Prisma-страницей; нельзя создавать второе независимое хранилище.
- Регрессионные тесты для существующего VLC launch path.

## Риски, выявленные аудитом

Главный риск — PotPlayer может переиспользовать существующий процесс, а generic spawn не даёт надёжной привязки к PID/window. Нельзя читать «первое окно PotPlayer» в системе: наблюдение должно быть связано с конкретной сессией и иметь диагностический статус при неоднозначности. Второй риск — текущая внешняя Prisma-страница может не предоставлять публичный метод сохранения прогресса; это будет проверено по загруженному runtime и фактическим вызовам Prisma. Третий риск — Win32 сообщения могут не проходить при разном уровне UAC, поэтому timeout и понятная ошибка обязательны.

## Следующий шаг

Перейти к подтверждению фактического PotPlayer IPC-контракта на Windows-совместимом bridge/harness, затем реализовать нативный слой и подключить его к существующему launch interception.
