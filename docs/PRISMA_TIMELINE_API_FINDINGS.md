# Prisma Timeline API findings

## Live bundle contract

The current official Prisma `app.min.js` exposes `Prisma.Timeline` with `update`, `view`, `watched` and `watchedEpisode`.

`Prisma.Player.play(data)` documents and receives `data.timeline` as a `Prisma.Timeline.view` object. That object includes `hash`, `percent`, `time`, `duration`, `profile`, `updated` and a `handler(percent, time, duration)` function. The built-in player saves progress through the timeline object's handler.

The live bundle's `watchedEpisode(card, season, episode)` computes the hash as:

```js
Prisma.Utils.hash([
  season,
  season > 10 ? ':' : '',
  episode,
  card.original_name || card.original_title,
].join(''))
```

This means an external player must prefer the already-created `data.timeline.hash` and its `handler`. Reconstructing the hash from guessed fields on the launch object is unsafe: a torrent/episode launch object may keep the card under a different property or omit `original_name`, and then the update can be written to a wrong hash or not associated with the visible episode.

`Prisma.Timeline.update({ hash, percent, time, duration })` writes to the `file_view[_profile]` storage, refreshes the visible timeline UI, emits `state:changed`, and sends premium account sync. `Prisma.Timeline.view(hash)` returns the same progress object and its handler.

## Diagnosis of episode screenshot

The screenshot shows episode 1 (`Кроличья нора`) at `Просмотрено 100%` with a complete blue line, while episode 2 (`Бойня`) has an empty grey line and no watched label. This is consistent with PotPlayer IPC working but the external session writing to a hash different from episode 2's real `data.timeline.hash`, or bypassing the exact timeline handler.

## Required fix

1. Prefer `data.timeline.hash` in `externalTimelineHash`.
2. Store `data.timeline` in the external session.
3. Call `data.timeline.handler(percent, time, duration)` when available; use `Prisma.Timeline.update` only as a compatibility fallback.
4. Publish the input timeline hash, data keys, episode/season fields, and save method in diagnostics so a real episode session can prove which card was updated.

## References

1. Official live Prisma bundle: `https://prisma.ws/app.min.js`
2. [PotPlayer x64 Function Library — AutoHotkey Community](https://www.autohotkey.com/boards/viewtopic.php?t=45385)
3. [PotPlayer command-line reference mirrored by VideoHelp](https://forum.videohelp.com/threads/406490-Potplayer-command-line-switches-not-working)
