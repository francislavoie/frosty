# Background playback when screen locked — design

**Date:** 2026-06-27
**Status:** Draft for review
**Scope:** Android (testable). iOS best-effort/unverified — see Non-goals.

## Goal

Add an opt-in setting that keeps the **audio** of a Twitch stream playing when the
screen is locked, instead of cutting out after a few seconds. When the screen is
off the stream drops to **audio-only** quality (no display to render), and the
previous video quality is restored when the screen turns back on.

Native-player only (`useNativePlayer`). The WebView player cannot play backgrounded
and is out of scope (it stays paused as today). Default **off** for backwards compat.

## Root cause of the current behavior

Audio currently plays for ~1–5 seconds when the screen locks, then stops. This is
**Android 17 background-audio hardening** enforcing a missing foreground service:

- Android 17 silently silences background audio (and fails audio-focus requests)
  unless the app runs a **`mediaPlayback` foreground service** started **while the
  app is visible** (to earn while-in-use capability).
- The forked native player (`tommyxchow/flutter-native-video-player`, `frosty-patches`)
  creates its ExoPlayer in `VideoPlayerView` (tied to the Activity) and has a
  `MediaSession` + MediaStyle notification (`VideoPlayerNotificationHandler`), **but
  no foreground service** — the plugin manifest declares only `WAKE_LOCK`, no
  `FOREGROUND_SERVICE*` permissions and no `<service>`.
- So Android grants a brief grace period, then suspends the process → audio stops.
  Worked on Android 16 (lenient), enforced on Android 17.

Refs: [Android 17 bg-audio hardening](https://developer.android.com/about/versions/17/changes/bg-audio),
[media3 MediaSessionService](https://developer.android.com/media/media3/session/background-playback).

## Non-goals

- **Keeping chat connected while locked.** Chat isn't visible when locked; it
  reconnects on unlock as today (and may stay connected as a side effect of the
  foreground service keeping the process alive). Not designed for explicitly.
- **iOS implementation/verification.** iOS already declares the `audio`
  `UIBackgroundModes`; background audio likely works or needs only minor audio-session
  work. Not actively implemented/tested here (no iOS test device). Tracked as follow-up.
- **WebView player background playback.** Not possible; out of scope.
- **Video playback while locked.** Screen off = no surface; audio only by definition.

## Architecture

Three components, each independently understandable:

### 1. Forked plugin — Android foreground media service (the core work)

Repo: `tommyxchow/flutter-native-video-player` (`frosty-patches`). This is a separate
repo from Frosty — see Risks for the logistics.

- **Manifest:** add `FOREGROUND_SERVICE` and `FOREGROUND_SERVICE_MEDIA_PLAYBACK`
  permissions; declare a `MediaSessionService` subclass with
  `android:foregroundServiceType="mediaPlayback"` and the
  `androidx.media3.session.MediaSessionService` intent-filter.
- **Service:** host the ExoPlayer + existing `MediaSession` so playback survives the
  Activity backgrounding. The service enters the foreground (posting the existing
  MediaStyle notification) while playback is active and background mode is enabled,
  and is **started while the app is visible** so Android 17 grants while-in-use
  capability. It stops/leaves foreground when playback ends or background mode is off.
  - The existing `VideoPlayerNotificationHandler` MediaSession/notification plumbing is
    reused — it just needs to be owned by / attached to the service rather than posted
    as a plain notification from the view.
- **Method-channel API:** add `setBackgroundPlaybackEnabled(bool)` (and any
  start/stop hooks) so Frosty controls whether the service engages. When disabled, the
  plugin behaves exactly as today (no FGS), preserving current behavior for the default-off case.

### 2. Frosty (Dart) — setting + screen-state-driven audio-only switch

- **Setting:** `backgroundPlayback` in `SettingsStore` (default `false`,
  `defaultBackgroundPlayback = false`), surfaced in `video_settings.dart` near the
  native-player toggle. Disabled/no-op when `useNativePlayer` is false.
- **Wiring:** when the setting is on and the native player is active, call the
  plugin's `setBackgroundPlaybackEnabled(true)`; off otherwise. Managed via a reaction
  in `NativeVideoStore`, mirroring the existing `_disposeAndroidAutoPipReaction` pattern.
- **Audio-only on screen-off:** observe **screen on/off** state (ACTION_SCREEN_OFF /
  ACTION_SCREEN_ON, surfaced from native via the plugin or a small platform channel).
  - On screen **off** (background playback enabled, playing, not user-paused): remember
    the current quality index and switch to the **audio-only** quality (reuses the
    existing audio-only path that already drops PiP and keeps audio).
  - On screen **on**: restore the remembered video quality.
  - Skip if already audio-only, user-paused, offline, or during an ad break.
  - Screen-state (not app-lifecycle) is the trigger so locking from the foreground *or*
    from PiP both behave correctly, and PiP/app-switching (screen on) keep video.

### 3. iOS (best-effort, unverified)

`UIBackgroundModes` already includes `audio`. Expected to keep audio alive when locked
with the audio session active. Verify the audio-only-on-lock switch and audio session
behavior on an iOS device in a follow-up; do not block the Android work on it.

## Behavior / data flow

| Event | Result |
|---|---|
| Screen locks, bg-playback ON, playing | FGS keeps player alive; switch to audio-only quality; audio continues |
| Screen unlocks | Restore prior video quality; video resumes |
| Press Home → PiP (screen on) | Unchanged: video in PiP window |
| Lock while in PiP | Screen off → audio-only; audio continues |
| bg-playback OFF (default) | Unchanged from today (no FGS); audio stops when locked |
| User explicitly paused, then locks | Stays paused (respect user intent); no FGS engaged |

## Edge cases

- **User-paused:** never force-resume; don't engage the FGS for a paused stream.
- **Already audio-only:** no quality save/restore; just keep playing.
- **Offline / ad break:** skip the audio-only switch (no stream / unreliable state).
- **FGS killed under memory pressure:** acceptable; on return the existing
  recovery/`handleAppResume` path restores playback (and we have the native rotation/
  resume fixes from prior work). No infinite re-start loop.
- **Toggle flipped mid-session:** reaction enables/disables the plugin background mode
  live; turning off while locked lets playback stop as before.

## Testing / verification

- **Unit (Dart):** the screen-off → save-quality / screen-on → restore-quality decision
  logic extracted into a small testable unit (mirrors `ChatInteractionPause` /
  `ChatLatencySync` precedent), covering: skip-when-user-paused, skip-when-already-audio-only,
  save/restore round-trip.
- **On-device (Android, manual):** with the setting on, lock the screen on a live
  stream → audio continues indefinitely (not 1–5s); unlock → video restores. Repeat
  from PiP. Confirm default-off still stops audio (no regression).
- **A17 hardening probe:** `adb shell cmd audio set-enable-hardening throw` and
  `adb shell dumpsys audio | grep AudioHardening` (expect a FGS with while-in-use, not
  `level: partial`).

## Risks / dependencies

- **Fork ownership:** the Android service work lives in `tommyxchow/flutter-native-video-player`,
  a separate repo. Implementing means either contributing upstream to that plugin or
  maintaining a further fork and pointing `pubspec.yaml` at it. Decide the path before
  implementation. The Frosty-side Dart work can proceed against the planned method-channel API.
- **Foreground-service notification UX:** Android requires a persistent notification
  while the FGS runs. The existing MediaStyle notification covers this; ensure it's not
  shown when background mode is off / not playing.
- **`ForegroundServiceNotStartInTime`:** media3 can throw this; mitigate per media3
  guidance (start FGS promptly while visible).
