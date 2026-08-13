# PotPlayer IPC re-audit

## Подтверждено независимыми источниками

The AutoHotkey PotPlayer Function Library documents the following `WM_USER` commands for the PotPlayer top-level window class `PotPlayer`: `0x5002` returns total time in milliseconds, `0x5004` returns current playback time in milliseconds, `0x5005` sets current time in milliseconds, and `0x5006` returns play status (`-1` stopped, `1` paused, `2` running). The same source explicitly uses `ahk_class PotPlayer` and `SendMessage` with `WM_USER` (`0x0400`).

The public `sid1552/PotPlayer-Resume` project independently describes the same two position queries, automatic polling every two seconds, `FindWindow("PotPlayer64", None)` in its sample, and `/seek=HH:MM:SS` for resume. It also warns that position tracking should be tested while PotPlayer is actually playing and that updates may intentionally wait several seconds.

Microsoft documents that `SendMessageTimeout` returns zero on failure/timeout and that messages at or above `WM_USER` are not system-marshalled between processes. This means the current use of a scalar `WPARAM`/`LPARAM` is appropriate for the documented scalar PotPlayer commands, but the implementation must expose whether the target window was found, which PID/class was selected, and whether each individual query returned a value. `EnumWindows` enumerates only top-level desktop windows on Windows 8+, which matches PotPlayer's main window discovery path.

## Current implementation risks

1. The current session passes the PID returned by `Command::spawn()` into `read_state`. PotPlayer may reuse/forward launch to another process or expose the main window under a different PID. If the selected top-level window PID differs, all IPC calls return `window_not_found`, even though PotPlayer is visible.
2. The current reader accepts any matching class and selects the first one. It does not return the full candidate window list or log each class/PID/title. A hidden/stale PotPlayer window can therefore be selected or the real window can be rejected by PID filtering.
3. The current reader requires the first `0x5004` query to succeed before querying duration/status. A transient response failure hides whether only current-time, duration, or status failed.
4. The current JSON only returns a coarse `ipc_timeout`/`window_not_found` reason. It does not expose the raw millisecond values, `WM_USER`, wParam, message result, window class, title, or selected PID, so a real Windows test cannot prove which stage failed.
5. Cross-process messages at `WM_USER` are not covered by Windows system marshalling. The current commands are scalar and do not pass pointers, so this is not itself a blocker, but any future command must not pass process-local pointers.
6. The source-level IPC codes are therefore probably correct; the highest-value test is now window/PID discovery plus per-message telemetry, followed by a live monotonicity check and a Prisma Timeline round trip.

## References

1. [PotPlayer x64 Function Library — AutoHotkey Community](https://www.autohotkey.com/boards/viewtopic.php?t=45385)
2. [sid1552/PotPlayer-Resume](https://github.com/sid1552/PotPlayer-Resume)
3. [Microsoft SendMessageTimeoutW documentation](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendmessagetimeoutw)
4. [Microsoft EnumWindows documentation](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-enumwindows)
