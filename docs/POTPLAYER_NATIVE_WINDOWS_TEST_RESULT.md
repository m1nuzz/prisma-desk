# PotPlayer native Windows IPC test result

## Scope

This evidence verifies the native PotPlayer side of Prisma Desk’s integration against the user-supplied `PotPlayerMini64.exe`/`PotPlayer64.dll` runtime on a real `windows-latest` GitHub Actions runner. It does **not** claim that Prisma account Timeline sync has been observed in a live authenticated Prisma session; that separate application-level gate remains exposed through `window.__desktop_external_progress_diagnostics`.

## Test run

The controlled native test used a local 74.933-second video-only MP4 fixture. It discovered a visible top-level `PotPlayer64` window, queried the documented scalar `WM_USER` commands from a separate process with `SendMessageTimeout`, observed six consecutive samples, then restarted PotPlayer with the production command-line `/seek=18` contract.

| Measurement | Recorded result |
|---|---:|
| GitHub Actions run | [31680974020](https://github.com/m1nuzz/prisma-desk/actions/runs/31680974020) |
| PotPlayer window class | `PotPlayer64` |
| Spawned/window PID match | `true` (`2500`) |
| Playback state | `2` / running |
| Media duration | `74.933 s` |
| Position samples | `7.416`, `9.419`, `11.417`, `13.416`, `15.424`, `17.419 s` |
| Requested resume position | `18.000 s` |
| First resumed position | `20.816 s` |
| Native result | `PASS` |

The position advanced by approximately two seconds between every two-second poll. The resumed position is valid under the test contract: it is later than the requested position because the player was intentionally allowed to load and continue playing before the first readback.

> **Confirmed:** the provided PotPlayer runtime creates a `PotPlayer64` window, returns changing duration/current-time/status values through `WM_USER` `0x5002`/`0x5004`/`0x5006`, and honours the `/seek` resume mechanism used by Prisma Desk.

## Remaining Prisma application gate

The running Prisma Desk app must still show `successfulReads >= 5`, `timelineUpdates >= 1`, a non-null `lastState.window`, and a `lastTimelineUpdate` in `window.__desktop_external_progress_diagnostics`. These fields are already implemented in the application. Passing them is the final proof that native PotPlayer values travelled through the Tauri bridge into `Prisma.Timeline.update`, rather than only into the local fallback store.

## Sources

1. [Successful native IPC workflow 31680974020](https://github.com/m1nuzz/prisma-desk/actions/runs/31680974020)
2. [PotPlayer x64 Function Library — AutoHotkey Community](https://www.autohotkey.com/boards/viewtopic.php?t=45385)
3. [PotPlayer command-line reference mirrored by VideoHelp](https://forum.videohelp.com/threads/406490-Potplayer-Command-line-switches-not-working)
4. [Microsoft SendMessageTimeoutW documentation](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendmessagetimeoutw)
