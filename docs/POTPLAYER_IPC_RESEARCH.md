# PotPlayer IPC research

## Подтверждённые сведения

| Область | Подтверждённый контракт | Надёжность |
|---|---|---|
| Запуск | `PotPlayerMini64.exe ["content"] [/switch]`; content может быть URL; `/seek=time` принимает `hh:mm:ss.ms` или секунды | Командная строка PotPlayer, опубликованная в справке программы и процитированная VideoHelp [1] |
| Окно | Практическая реализация PotPlayer-Resume использует `FindWindow("PotPlayer64", None)` | Рабочая сторонняя реализация [2]; нужно дополнительно привязать окно к PID при реализации |
| Общий Win32 message | `WM_USER` = `0x0400` | Подтверждено примером AutoHotkey [3] |
| Длительность | `wParam=0x5002`, `lParam=0`, результат в миллисекундах | Подтверждено примером AutoHotkey и PotPlayer-Resume [2] [3] |
| Текущая позиция | `wParam=0x5004`, `lParam=0`, результат в миллисекундах | Подтверждено примером AutoHotkey и PotPlayer-Resume [2] [3] |
| Установка позиции | `wParam=0x5005`, `lParam=milliseconds` | Подтверждено примером AutoHotkey [3]; использовать как fallback после `/seek` |
| Состояние | `wParam=0x5006`: `-1` stopped, `1` paused, `2` running | Подтверждено примером AutoHotkey [3] |
| Период опроса | Потенциально каждые 2 секунды; запись только после фактического движения/минимального времени | Реализовано в PotPlayer-Resume [2], но интервалы должны оставаться конфигурируемыми в Prisma |
| Timeout | Использовать `SendMessageTimeoutW`; не блокировать Tauri/main thread. Неудача и timeout возвращают «нет показаний», не ноль | Microsoft API documentation [4] |

## Важная корректировка предыдущей гипотезы

В первоначальном плане код `0x5002` был указан как предварительный код позиции. После чтения полных источников установлено: **`0x5002` — длительность, `0x5004` — текущий таймкод**. Реализация должна использовать `0x5004` для resume/progress и `0x5002` для duration.

## Ограничения источников

Команды `0x5002`–`0x5006` происходят из community reverse-engineering/automation examples, а не из стабильной публичной официальной SDK-документации PotPlayer. Поэтому код должен держать их в одном Windows-only модуле, проверять диапазон и монотонность результатов, а актуальный PotPlayer x64 необходимо прогнать через smoke test на Windows 11 перед release.

Официальная/практическая командная строка допускает `/new`, поэтому запуск новой сессии следует делать с `/new`, если это совместимо с текущей политикой PotPlayer; без этого PotPlayer может переиспользовать существующий процесс. Для потоков также нужно сохранить query-параметры URL и не использовать shell string.

## Источники

[1]: https://forum.videohelp.com/threads/406490-Potplayer-Command-line-switches-not-working — VideoHelp: Поток командной строки PotPlayer, включая `/seek`, `/new`, `/current` и URL.
[2]: https://github.com/sid1552/PotPlayer-Resume — sid1552/PotPlayer-Resume: практическая утилита автоопределения окна, polling WM_USER и resume `/seek`.
[3]: https://www.autohotkey.com/boards/viewtopic.php?t=45385 — AutoHotkey Community: PotPlayer x64 Function Library с кодами `0x5002`–`0x5006`.
[4]: https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendmessagetimeoutw — Microsoft Learn: сигнатура, timeout semantics и ограничения `SendMessageTimeoutW`.
