# PotPlayer / Prisma Desk end-to-end test plan

## Цель

Тест должен доказать не факт запуска `PotPlayerMini64.exe`, а весь путь данных: PotPlayer реально проигрывает медиа, внешний процесс получает изменяющийся таймкод, Prisma Desk сохраняет его в Prisma Timeline, а повторный запуск возвращается к сохранённой позиции.

## Stage A — direct PotPlayer IPC gate

The PowerShell diagnostic starts PotPlayer with a deterministic local media file or a stable test URL and enumerates every visible top-level window whose class is `PotPlayer` or `PotPlayer64`. It records the spawned PID, every candidate window PID, class and title, and the selected window. The test does not require the spawned PID to equal the window PID; it first tries an exact match and then applies a documented PotPlayer-window fallback.

For the selected window it sends scalar `WM_USER` messages with `SendMessageTimeout`: `0x5002` for duration, `0x5004` for current position and `0x5006` for status. Each result is logged independently with the raw value, elapsed time and error/reason. The gate passes only when duration is positive, status is `2` (running), and at least five consecutive position samples are finite and non-decreasing with at least two distinct values. Resume is tested through a separate PotPlayer restart with the same command-line `/seek=<seconds>` contract used by Prisma Desk; direct `0x5005` live seeking is intentionally not a release requirement.

This stage catches the current failure class directly: a visible PotPlayer window with no matching spawned PID, wrong class selection, a timeout for one particular message, or a non-changing position.

## Stage B — Prisma bridge gate

Start the Windows Prisma Desk build, open a known film from Prisma with PotPlayer selected, and inspect the exposed diagnostic snapshot:

```js
window.__desktop_external_progress_diagnostics
```

The snapshot must report the selected PID, the actual window PID, at least five successful reads, changing `positionSec`, a positive `durationSec`, no persistent `window_not_found` or `ipc_timeout`, and at least one `timelineUpdate` whose hash, time, duration and percent are recorded. The same snapshot is also written to `localStorage` under `prisma_desktop_external_progress_diagnostics_v1` for post-run inspection.

## Stage C — resume round trip

1. Clear the test item’s previous progress and start the known media through Prisma Desk.
2. Let PotPlayer play until the diagnostic position is at least 20 seconds, then stop/close the external session.
3. Verify that the diagnostics contain a final flush and that `Prisma.Timeline.view(hash).time` is within 1.5 seconds of the last PotPlayer position.
4. Start the same Prisma item again. The diagnostic `resumePositionSec` and PotPlayer’s first readable position must be no earlier than 1.5 seconds before the saved time and no later than nine seconds after it, allowing for real startup latency plus the first seconds of active playback.
5. Seek to at least 92% or leave less than ten seconds, stop and reopen. The diagnostic must mark `watched=true`, write Timeline percent `100`, and resume from zero on the next launch.

## PASS criteria

| Gate | Required evidence |
|---|---|
| Direct IPC | Five changing position samples, positive duration and status `2`. |
| Window selection | Candidate window list is non-empty; selected class is `PotPlayer` or `PotPlayer64`; actual window PID is recorded even if it differs from spawned PID. |
| Prisma bridge | At least five successful reads and no persistent not-found/timeout state. |
| Timeline | `Prisma.Timeline.update` telemetry contains the same hash, time and duration as the PotPlayer sample within tolerance. |
| Resume | Second launch uses `/seek`; first readable position is within `[saved - 1.5 s, saved + 9 s]`. |
| Watched | Near-end playback writes percent `100`, and the next launch starts at zero. |

A result is **not** considered verified if only the process opens, if a single IPC response is observed, or if the test relies solely on a local fallback record without a matching Prisma Timeline update.

## Environment requirements

Use Windows 11, the same UAC elevation level for Prisma Desk and PotPlayer, a visible non-minimized PotPlayer window, and a local deterministic media file with duration longer than 60 seconds. Network streams can be tested later, but they are not suitable as the first gate because redirects, HLS buffering and unknown duration can make a correct IPC reader appear broken.

## References

1. [PotPlayer x64 Function Library — AutoHotkey Community](https://www.autohotkey.com/boards/viewtopic.php?t=45385)
2. [sid1552/PotPlayer-Resume](https://github.com/sid1552/PotPlayer-Resume)
3. [Microsoft SendMessageTimeoutW documentation](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendmessagetimeoutw)
4. [Microsoft EnumWindows documentation](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumwindows)
