# Background Playback — Plugin PR Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.
>
> **Repo:** This plan is implemented in the **plugin fork**, not Frosty:
> `~/repos/flutter-native-video-player` (origin = `francislavoie/flutter-native-video-player`,
> upstream = `tommyxchow/flutter-native-video-player`), branch **`background-playback`**
> (based on the pinned ref `6460859`). Frosty's `pubspec.yaml` already points at this
> branch. The end goal is a PR to `tommyxchow/flutter-native-video-player`.

**Goal:** Keep the native player's audio alive when the app is backgrounded / screen-locked on Android by running a `mediaPlayback` foreground service, and expose two APIs Frosty consumes: `setBackgroundPlaybackEnabled(bool)` and a screen-state stream.

**Architecture:** Lighter than a full `MediaSessionService` rewrite. The ExoPlayer stays where it is (`SharedPlayerManager`/`VideoPlayerView`); a new `PlaybackForegroundService` calls `startForeground()` with the **existing** `VideoPlayerNotificationHandler` MediaStyle notification, keeping the process (and thus audio) alive. The service is started **while the app is visible** (when Frosty enables background playback) so Android 17 grants while-in-use capability. A `BroadcastReceiver` for `ACTION_SCREEN_ON/OFF` feeds a Dart `screenStateStream`.

**Tech Stack:** Kotlin, AndroidX media3 (already a dependency), Flutter platform channels.

## Global Constraints

- **Android 17 compliance:** the FGS must be type `mediaPlayback` and **started while the app is visible** (Frosty calls `setBackgroundPlaybackEnabled(true)` from the foreground), or audio is silently silenced. (spec)
- **Default-off / opt-in:** the service only runs when Frosty enables it; with it disabled the plugin behaves exactly as today (no FGS, no behavior change). (spec)
- **media3** is already on the classpath (`media3-session`, `media3-exoplayer`); reuse the existing `MediaSession` in `VideoPlayerNotificationHandler` — do not create a second session. (verified)
- **Reuse `NOTIFICATION_ID = 1001`** and the existing notification channel so there is exactly one media notification. (verified)
- iOS is **best-effort/unverified** (no test device) — it already declares the `audio` background mode; see the iOS note. Do not block the Android PR on it.

## Consumer contract (must match the Frosty plan's "Consumed interface")

- `NativeVideoPlayerController.setBackgroundPlaybackEnabled(bool enabled)`
- `NativeVideoPlayerController.screenStateStream` → `Stream<bool>` (`false` = screen off, `true` = screen on)

## File Structure (in the plugin repo)

- Modify: `android/src/main/AndroidManifest.xml` — FGS permissions + `<service>`.
- Create: `android/.../PlaybackForegroundService.kt` — the foreground service.
- Create: `android/.../handlers/ScreenStateReceiver.kt` — `ACTION_SCREEN_ON/OFF` receiver → EventChannel sink.
- Modify: `android/.../handlers/VideoPlayerMethodHandler.kt` — `setBackgroundPlaybackEnabled` dispatch.
- Modify: `android/.../NativeVideoPlayerPlugin.kt` — register the screen-state `EventChannel`, own the receiver lifecycle.
- Modify: `android/.../manager/SharedPlayerManager.kt` — expose the active controller's notification + player for the service.
- Modify: `lib/src/platform/video_player_method_channel.dart` + the platform interface + `lib/.../native_video_player_controller.dart` — add the two Dart APIs.
- iOS: note only (no code in this plan).

---

### Task 1: Manifest — permissions + service declaration

**Files:** Modify `android/src/main/AndroidManifest.xml`

**Interfaces:** Produces the `PlaybackForegroundService` registration (class created in Task 2).

- [ ] **Step 1: Add permissions + service**

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

    <application>
        <service
            android:name=".PlaybackForegroundService"
            android:exported="false"
            android:foregroundServiceType="mediaPlayback" />
    </application>
</manifest>
```

- [ ] **Step 2: Build the plugin to verify the manifest merges**

Run: `git -C ~/repos/flutter-native-video-player status` then build Frosty (`~/run-frosty.sh`) — manifest-merger failures surface at the Gradle `processDebugMainManifest` task.
Expected: build proceeds past manifest merge (the service class is added in Task 2; until then keep the `<service>` line commented if Gradle validates the class exists — uncomment in Task 2).

- [ ] **Step 3: Commit**

```bash
git -C ~/repos/flutter-native-video-player add android/src/main/AndroidManifest.xml
git -C ~/repos/flutter-native-video-player commit -m "feat(android): declare mediaPlayback foreground service + permissions"
```

---

### Task 2: `PlaybackForegroundService`

**Files:** Create `android/src/main/kotlin/com/huddlecommunity/better_native_video_player/PlaybackForegroundService.kt`

**Interfaces:**
- Consumes: `SharedPlayerManager.activeForegroundNotification(): Pair<Int, Notification>?` (added in Task 4).
- Produces: `PlaybackForegroundService.start(context)` / `stop(context)` companion helpers.

- [ ] **Step 1: Implement the service**

```kotlin
package com.huddlecommunity.better_native_video_player

import android.app.Service
import android.content.Context
import android.content.Intent
import android.os.IBinder
import android.util.Log
import com.huddlecommunity.better_native_video_player.manager.SharedPlayerManager

/**
 * Foreground service of type mediaPlayback. Holds the process alive so the
 * shared ExoPlayer keeps decoding audio when the app is backgrounded / screen
 * locked (Android 17 silences background audio without this). Reuses the
 * existing MediaSession notification so there is exactly one media notification.
 *
 * Started by [setBackgroundPlaybackEnabled] *while the app is visible* so the
 * FGS is granted while-in-use capability.
 */
class PlaybackForegroundService : Service() {
    companion object {
        private const val TAG = "PlaybackFgService"

        fun start(context: Context) {
            val intent = Intent(context, PlaybackForegroundService::class.java)
            context.startForegroundService(intent)
        }

        fun stop(context: Context) {
            context.stopService(Intent(context, PlaybackForegroundService::class.java))
        }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val active = SharedPlayerManager.activeForegroundNotification()
        if (active == null) {
            // No active player/notification to foreground — nothing to keep alive.
            Log.w(TAG, "No active notification; stopping service")
            stopSelf()
            return START_NOT_STICKY
        }
        val (notificationId, notification) = active
        startForeground(notificationId, notification)
        return START_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

- [ ] **Step 2: Uncomment the `<service>` in the manifest (if commented in Task 1) and build**

Run: `~/run-frosty.sh`
Expected: compiles; service class resolves.

- [ ] **Step 3: Commit**

```bash
git -C ~/repos/flutter-native-video-player add android/src/main/kotlin/com/huddlecommunity/better_native_video_player/PlaybackForegroundService.kt
git -C ~/repos/flutter-native-video-player commit -m "feat(android): add PlaybackForegroundService reusing the media notification"
```

---

### Task 3: `SharedPlayerManager.activeForegroundNotification()`

**Files:** Modify `android/.../manager/SharedPlayerManager.kt`, `android/.../handlers/VideoPlayerNotificationHandler.kt`

**Interfaces:**
- Produces: `SharedPlayerManager.activeForegroundNotification(): Pair<Int, Notification>?` and `VideoPlayerNotificationHandler.NOTIFICATION_ID` (expose the existing private const) + `VideoPlayerNotificationHandler.buildForegroundNotification(): Notification?`.

- [ ] **Step 1: Expose the notification from the handler**

In `VideoPlayerNotificationHandler`, make `NOTIFICATION_ID` accessible (move to `companion object` public, or add a getter) and add a public method that returns the current MediaStyle notification (reusing the existing private `buildNotification()`), or null if the session isn't ready:

```kotlin
    companion object {
        const val NOTIFICATION_ID = 1001 // was private
        // ...
    }

    /** The current MediaStyle notification for use as a foreground-service
     *  notification, or null if the MediaSession isn't initialized yet. */
    fun foregroundNotification(): Notification? =
        if (mediaSession != null) buildNotification() else null
```

- [ ] **Step 2: Add the manager accessor**

In `SharedPlayerManager`, add (it already holds `notificationHandlers` keyed by controllerId):

```kotlin
    /** The (id, notification) for any active player, for the foreground service.
     *  Returns the first ready notification; null if none. */
    fun activeForegroundNotification(): Pair<Int, android.app.Notification>? {
        for (handler in notificationHandlers.values) {
            val n = handler.foregroundNotification() ?: continue
            return VideoPlayerNotificationHandler.NOTIFICATION_ID to n
        }
        return null
    }
```

- [ ] **Step 3: Build to verify it compiles**

Run: `~/run-frosty.sh`
Expected: compiles.

- [ ] **Step 4: Commit**

```bash
git -C ~/repos/flutter-native-video-player add android/src/main/kotlin/com/huddlecommunity/better_native_video_player/manager/SharedPlayerManager.kt android/src/main/kotlin/com/huddlecommunity/better_native_video_player/handlers/VideoPlayerNotificationHandler.kt
git -C ~/repos/flutter-native-video-player commit -m "feat(android): expose the media notification for the foreground service"
```

---

### Task 4: `setBackgroundPlaybackEnabled` method channel

**Files:** Modify `android/.../handlers/VideoPlayerMethodHandler.kt`; Dart `lib/src/platform/video_player_method_channel.dart`, the platform interface, and the public controller.

**Interfaces:**
- Consumes: `PlaybackForegroundService.start/stop` (Task 2).
- Produces: Dart `NativeVideoPlayerController.setBackgroundPlaybackEnabled(bool)`.

- [ ] **Step 1: Android — handle the method**

In `VideoPlayerMethodHandler.handleMethodCall`'s `when (call.method)` block add:

```kotlin
            "setBackgroundPlaybackEnabled" -> handleSetBackgroundPlaybackEnabled(call, result)
```

And the handler (uses the plugin/application context; starts the FGS *now*, i.e. while visible, per A17):

```kotlin
    private fun handleSetBackgroundPlaybackEnabled(call: MethodCall, result: MethodChannel.Result) {
        val enabled = call.argument<Boolean>("enabled") ?: false
        val context = NativeVideoPlayerPlugin.applicationContext
        if (context == null) {
            result.error("no_context", "Plugin not attached", null)
            return
        }
        if (enabled) {
            PlaybackForegroundService.start(context)
        } else {
            PlaybackForegroundService.stop(context)
        }
        result.success(null)
    }
```

(Expose `NativeVideoPlayerPlugin.applicationContext` as a companion `var` set in `onAttachedToEngine` / cleared in `onDetachedFromEngine`.)

- [ ] **Step 2: Dart — method-channel + controller API**

In `video_player_method_channel.dart` (mirroring `enableAutomaticInlinePip`):

```dart
  @override
  Future<void> setBackgroundPlaybackEnabled(bool enabled) async {
    await _methodChannel.invokeMethod<void>(
      'setBackgroundPlaybackEnabled',
      <String, Object>{'enabled': enabled},
    );
  }
```

Add `setBackgroundPlaybackEnabled(bool)` to the platform interface and the public `NativeVideoPlayerController` (delegating to the platform), matching the existing PiP method pattern.

- [ ] **Step 3: Build to verify it compiles**

Run: `~/run-frosty.sh`
Expected: compiles; `controller.setBackgroundPlaybackEnabled(true/false)` is callable from Frosty.

- [ ] **Step 4: Commit**

```bash
git -C ~/repos/flutter-native-video-player add -A
git -C ~/repos/flutter-native-video-player commit -m "feat: add setBackgroundPlaybackEnabled to start/stop the foreground service"
```

---

### Task 5: Screen-state EventChannel

**Files:** Create `android/.../handlers/ScreenStateReceiver.kt`; modify `NativeVideoPlayerPlugin.kt`; Dart `video_player_method_channel.dart` + interface + controller (`screenStateStream`).

**Interfaces:**
- Produces: Dart `NativeVideoPlayerController.screenStateStream` → `Stream<bool>`.

- [ ] **Step 1: Android — BroadcastReceiver bridging to an EventChannel sink**

```kotlin
package com.huddlecommunity.better_native_video_player.handlers

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import io.flutter.plugin.common.EventChannel

/** Emits false on ACTION_SCREEN_OFF, true on ACTION_SCREEN_ON, to the screen-state EventChannel. */
class ScreenStateReceiver(private val sink: EventChannel.EventSink) : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        when (intent?.action) {
            Intent.ACTION_SCREEN_OFF -> sink.success(false)
            Intent.ACTION_SCREEN_ON -> sink.success(true)
        }
    }
}
```

- [ ] **Step 2: Android — register an EventChannel + the receiver in the plugin**

In `NativeVideoPlayerPlugin.onAttachedToEngine`, register an `EventChannel("native_video_player/screen_state")` whose `onListen` registers a `ScreenStateReceiver` for an `IntentFilter` of `ACTION_SCREEN_ON`/`ACTION_SCREEN_OFF`, and `onCancel` unregisters it. Unregister in `onDetachedFromEngine`.

```kotlin
        EventChannel(binding.binaryMessenger, "native_video_player/screen_state")
            .setStreamHandler(object : EventChannel.StreamHandler {
                override fun onListen(args: Any?, sink: EventChannel.EventSink) {
                    screenStateReceiver = ScreenStateReceiver(sink)
                    val filter = IntentFilter().apply {
                        addAction(Intent.ACTION_SCREEN_ON)
                        addAction(Intent.ACTION_SCREEN_OFF)
                    }
                    applicationContext?.registerReceiver(screenStateReceiver, filter)
                }
                override fun onCancel(args: Any?) {
                    screenStateReceiver?.let { applicationContext?.unregisterReceiver(it) }
                    screenStateReceiver = null
                }
            })
```

- [ ] **Step 3: Dart — expose `screenStateStream`**

In `video_player_method_channel.dart`:

```dart
  static const _screenStateChannel = EventChannel('native_video_player/screen_state');

  @override
  Stream<bool> get screenStateStream =>
      _screenStateChannel.receiveBroadcastStream().map((e) => e as bool);
```

Add `Stream<bool> get screenStateStream` to the platform interface and the public controller.

- [ ] **Step 4: Build + on-device verification**

Run: `~/run-frosty.sh` (with the Frosty-plan Task 3 wiring present, or a temporary log listener).
Expected: locking/unlocking the device emits `false`/`true` on the stream.

- [ ] **Step 5: Commit**

```bash
git -C ~/repos/flutter-native-video-player add -A
git -C ~/repos/flutter-native-video-player commit -m "feat: add screen-state EventChannel (ACTION_SCREEN_ON/OFF)"
```

---

### Task 6: End-to-end verification + push + PR

- [ ] **Step 1: Push the branch**

```bash
git -C ~/repos/flutter-native-video-player push origin background-playback
```

- [ ] **Step 2: Re-resolve Frosty against the latest branch commit**

Run (in Frosty): `flutter pub upgrade better_native_video_player`
Expected: `pubspec.lock` `resolved-ref` advances to the new branch HEAD.

- [ ] **Step 3: On-device test (Android)**

Complete the Frosty plan's Task 3 wiring, then run the spec's Verification checklist:
1. Native player + background-playback toggle on; lock screen → audio continues indefinitely (not 1–5s); quality drops to audio-only.
2. Unlock → video quality restored.
3. `adb shell dumpsys audio | grep AudioHardening` shows a FGS with while-in-use (not `level: partial`).
4. Toggle off → audio stops on lock (no regression).

- [ ] **Step 4: Open the PR upstream**

```bash
gh pr create --repo tommyxchow/flutter-native-video-player \
  --head francislavoie:background-playback \
  --title "Background audio: mediaPlayback foreground service + screen-state events" \
  --body "Adds a mediaPlayback foreground service (Android 17 compliant) reusing the existing MediaSession notification, a setBackgroundPlaybackEnabled method, and an ACTION_SCREEN_ON/OFF EventChannel. Keeps live-stream audio alive when the screen is locked."
```

- [ ] **Step 5: After merge, repoint Frosty's pubspec**

Revert `pubspec.yaml` to `url: https://github.com/tommyxchow/flutter-native-video-player.git` at the merged `frosty-patches` (or merged commit), then `flutter pub get`.

## iOS note (out of scope, best-effort)

iOS already declares `UIBackgroundModes → audio`; the existing audio session likely
keeps audio alive when locked. If iOS background playback is pursued: implement
`setBackgroundPlaybackEnabled` (a no-op or audio-session category assertion) and a
screen-state stream via `UIApplication` `protectedDataWillBecomeUnavailable` /
`...DidBecomeAvailable` or display notifications, mirroring the Android contract.
Verify on an iOS device (none available now).

## Native testing note

The plugin has minimal unit-test scaffolding and these components (foreground
service, BroadcastReceiver, EventChannel) are integration-level. Verification is by
build + on-device behavior (Task 6 / spec checklist), not red-green unit tests.
Keep the Frosty-side decision logic (`ScreenLockAudioPolicy`) as the unit-tested seam.
