# Background Playback — Frosty Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a default-off `backgroundPlayback` setting and the Dart-side logic that, on the native player, switches the stream to audio-only when the screen turns off and restores the prior video quality when it turns back on.

**Architecture:** Pure-Dart work in the Frosty repo. A small unit-tested decision class (`ScreenLockAudioPolicy`) decides when to drop to / restore from audio-only. `NativeVideoStore` wires that policy to a screen-state stream and a background-enable toggle exposed by the native player plugin. All native Android work (the `mediaPlayback` foreground service, the screen-state events, and the `setBackgroundPlaybackEnabled` method) is delivered by the separate **plugin PR plan** — this plan consumes that contract.

**Tech Stack:** Flutter, MobX (codegen), `better_native_video_player` (forked plugin), `flutter_test`.

## Global Constraints

- Native-player only; the setting is a no-op on the WebView player. (spec)
- Default **off** (`defaultBackgroundPlayback = false`). (spec)
- Package imports (`package:frosty/...`), single quotes, trailing commas always. (AGENTS.md)
- Never edit `.g.dart` files by hand — regenerate with `dart run build_runner build --delete-conflicting-outputs`. (AGENTS.md)
- Android is the target; iOS is out of scope for this plan. (spec)

## Consumed interface (delivered by the plugin PR plan)

These are implemented in the `tommyxchow/flutter-native-video-player` PR; this plan
writes Dart against them. Task 3's end-to-end run depends on the plugin branch being
available via `pubspec.yaml`'s `ref`; Tasks 1–2 are fully testable without it.

- `NativeVideoPlayerController.setBackgroundPlaybackEnabled(bool enabled)` — engages/disengages the foreground media service.
- `NativeVideoPlayerController.screenStateStream` → `Stream<bool>` — emits `false` when the screen turns off, `true` when it turns on.

## File Structure

- Create: `lib/screens/channel/video/screen_lock_audio_policy.dart` — pure decision logic (no Flutter/MobX deps).
- Create: `test/screens/channel/video/screen_lock_audio_policy_test.dart` — unit tests.
- Modify: `lib/screens/settings/stores/settings_store.dart` — add `backgroundPlayback` observable + default + reset.
- Regenerate: `lib/screens/settings/stores/settings_store.g.dart` (via build_runner).
- Modify: `lib/screens/settings/video_settings.dart` — add the toggle under the native-player section.
- Modify: `lib/screens/channel/video/native_video_store.dart` — own a `ScreenLockAudioPolicy`, subscribe to the screen-state stream, toggle plugin background mode, switch/restore quality.
- Test: `test/screens/settings/background_playback_setting_test.dart` — setting default/round-trip.

---

### Task 1: `backgroundPlayback` setting

**Files:**
- Modify: `lib/screens/settings/stores/settings_store.dart`
- Regenerate: `lib/screens/settings/stores/settings_store.g.dart`
- Test: `test/screens/settings/background_playback_setting_test.dart`

**Interfaces:**
- Produces: `SettingsStore.backgroundPlayback` (observable `bool`, default `false`), `SettingsStore.defaultBackgroundPlayback`.

- [ ] **Step 1: Write the failing test**

```dart
// test/screens/settings/background_playback_setting_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:frosty/screens/settings/stores/settings_store.dart';

void main() {
  test('backgroundPlayback defaults to false', () {
    final store = SettingsStore.fromJson({});
    expect(store.backgroundPlayback, isFalse);
  });

  test('backgroundPlayback round-trips through json', () {
    final store = SettingsStore.fromJson({})..backgroundPlayback = true;
    final restored = SettingsStore.fromJson(store.toJson());
    expect(restored.backgroundPlayback, isTrue);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/screens/settings/background_playback_setting_test.dart`
Expected: FAIL — `backgroundPlayback` getter not defined on `SettingsStore`.

- [ ] **Step 3: Add the default constant**

In `lib/screens/settings/stores/settings_store.dart`, next to `defaultUseNativePlayer`:

```dart
  static const defaultUseNativePlayer = false;
  static const defaultBackgroundPlayback = false;
```

- [ ] **Step 4: Add the observable**

Directly below the `useNativePlayer` field:

```dart
  @JsonKey(defaultValue: defaultUseNativePlayer)
  @observable
  var useNativePlayer = defaultUseNativePlayer;

  /// Keep audio playing when the screen is locked (native player only).
  @JsonKey(defaultValue: defaultBackgroundPlayback)
  @observable
  var backgroundPlayback = defaultBackgroundPlayback;
```

- [ ] **Step 5: Add to the reset method**

Find the `reset()` method (where `useNativePlayer = defaultUseNativePlayer;` is reset) and add:

```dart
    useNativePlayer = defaultUseNativePlayer;
    backgroundPlayback = defaultBackgroundPlayback;
```

- [ ] **Step 6: Regenerate codegen**

Run: `dart run build_runner build --delete-conflicting-outputs`
Expected: writes `settings_store.g.dart` with the new `_$backgroundPlayback*` atom and json keys.

- [ ] **Step 7: Run test to verify it passes**

Run: `flutter test test/screens/settings/background_playback_setting_test.dart`
Expected: PASS (both tests).

- [ ] **Step 8: Commit**

```bash
git add lib/screens/settings/stores/settings_store.dart lib/screens/settings/stores/settings_store.g.dart test/screens/settings/background_playback_setting_test.dart
git commit -m "feat(settings): add backgroundPlayback setting (default off)"
```

---

### Task 2: `ScreenLockAudioPolicy` decision logic

**Files:**
- Create: `lib/screens/channel/video/screen_lock_audio_policy.dart`
- Test: `test/screens/channel/video/screen_lock_audio_policy_test.dart`

**Interfaces:**
- Produces:
  - `ScreenLockAudioPolicy.onScreenOff({required bool backgroundPlaybackEnabled, required bool playing, required bool userPaused, required int currentQualityIndex, required int? audioOnlyQualityIndex, required bool offline, required bool adBreakActive}) → int?` — returns the audio-only index to switch to (and remembers `currentQualityIndex`), or `null` for no-op.
  - `ScreenLockAudioPolicy.onScreenOn() → int?` — returns the saved index to restore (then clears it), or `null`.
  - `ScreenLockAudioPolicy.reset() → void`.

- [ ] **Step 1: Write the failing test**

```dart
// test/screens/channel/video/screen_lock_audio_policy_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:frosty/screens/channel/video/screen_lock_audio_policy.dart';

void main() {
  late ScreenLockAudioPolicy policy;
  setUp(() => policy = ScreenLockAudioPolicy());

  int? off({
    bool enabled = true,
    bool playing = true,
    bool userPaused = false,
    int current = 2,
    int? audioOnly = 5,
    bool offline = false,
    bool ad = false,
  }) => policy.onScreenOff(
        backgroundPlaybackEnabled: enabled,
        playing: playing,
        userPaused: userPaused,
        currentQualityIndex: current,
        audioOnlyQualityIndex: audioOnly,
        offline: offline,
        adBreakActive: ad,
      );

  test('switches to the audio-only index and restores the prior index', () {
    expect(off(current: 2, audioOnly: 5), 5);
    expect(policy.onScreenOn(), 2);
  });

  test('onScreenOn returns null once consumed', () {
    off();
    policy.onScreenOn();
    expect(policy.onScreenOn(), isNull);
  });

  test('no-op when disabled', () => expect(off(enabled: false), isNull));
  test('no-op when paused', () => expect(off(playing: false), isNull));
  test('no-op when user paused', () => expect(off(userPaused: true), isNull));
  test('no-op when offline', () => expect(off(offline: true), isNull));
  test('no-op during ad break', () => expect(off(ad: true), isNull));
  test('no-op when no audio-only quality', () => expect(off(audioOnly: null), isNull));
  test('no-op when already audio-only', () => expect(off(current: 5, audioOnly: 5), isNull));

  test('onScreenOn returns null if screen-off was a no-op', () {
    off(enabled: false);
    expect(policy.onScreenOn(), isNull);
  });

  test('reset clears a pending restore', () {
    off();
    policy.reset();
    expect(policy.onScreenOn(), isNull);
  });
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `flutter test test/screens/channel/video/screen_lock_audio_policy_test.dart`
Expected: FAIL — `ScreenLockAudioPolicy` not defined.

- [ ] **Step 3: Write the implementation**

```dart
// lib/screens/channel/video/screen_lock_audio_policy.dart

/// Decides when the native player should drop to audio-only because the screen
/// turned off, and what video quality to restore when it turns back on.
///
/// Pure logic with no Flutter/MobX dependency so it can be unit-tested without
/// constructing the heavyweight video store (mirrors ChatLatencySync).
class ScreenLockAudioPolicy {
  int? _savedQualityIndex;

  /// The screen turned off. Returns the audio-only quality index to switch to
  /// (remembering the current index for restore), or null if nothing should
  /// change — disabled, not actively playing, user-paused, offline, mid ad
  /// break, no audio-only quality available, or already audio-only.
  int? onScreenOff({
    required bool backgroundPlaybackEnabled,
    required bool playing,
    required bool userPaused,
    required int currentQualityIndex,
    required int? audioOnlyQualityIndex,
    required bool offline,
    required bool adBreakActive,
  }) {
    if (!backgroundPlaybackEnabled ||
        !playing ||
        userPaused ||
        offline ||
        adBreakActive) {
      return null;
    }
    if (audioOnlyQualityIndex == null ||
        currentQualityIndex == audioOnlyQualityIndex) {
      return null;
    }
    _savedQualityIndex = currentQualityIndex;
    return audioOnlyQualityIndex;
  }

  /// The screen turned on. Returns the quality index to restore (then clears
  /// it), or null if the screen-off didn't switch.
  int? onScreenOn() {
    final saved = _savedQualityIndex;
    _savedQualityIndex = null;
    return saved;
  }

  /// Drop any pending restore (e.g. on dispose or a hard refresh).
  void reset() => _savedQualityIndex = null;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `flutter test test/screens/channel/video/screen_lock_audio_policy_test.dart`
Expected: PASS (all cases).

- [ ] **Step 5: Commit**

```bash
git add lib/screens/channel/video/screen_lock_audio_policy.dart test/screens/channel/video/screen_lock_audio_policy_test.dart
git commit -m "feat(video): add ScreenLockAudioPolicy decision logic"
```

---

### Task 3: Wire the policy into `NativeVideoStore`

> **Dependency:** consumes `setBackgroundPlaybackEnabled` and `screenStateStream` from the plugin PR plan. Build against the plugin PR branch (`pubspec.yaml` `ref`). Unit coverage lives in Tasks 1–2; this task's verification is on-device/manual (see Verification).

**Files:**
- Modify: `lib/screens/channel/video/native_video_store.dart`
- Modify: `lib/screens/settings/video_settings.dart`

**Interfaces:**
- Consumes: `SettingsStore.backgroundPlayback` (Task 1); `ScreenLockAudioPolicy` (Task 2); plugin `setBackgroundPlaybackEnabled(bool)` and `screenStateStream` (plugin plan).

- [ ] **Step 1: Add the policy field + screen-state subscription field**

In `NativeVideoStoreBase`, near the other private fields (`_pip`, `_chatLatencySync`):

```dart
  final _screenLockAudioPolicy = ScreenLockAudioPolicy();
  StreamSubscription<bool>? _screenStateSub;
  ReactionDisposer? _disposeBackgroundPlaybackReaction;
```

Add the import at the top:

```dart
import 'package:frosty/screens/channel/video/screen_lock_audio_policy.dart';
```

- [ ] **Step 2: Add a helper to find the audio-only quality index**

The audio-only quality has `width == 0 && height == 0` (see the existing detection in `_applyBackgroundPipForQuality`). Index 0 is "Auto"; quality objects start at index 1.

```dart
  /// Index into [_availableStreamQualities] of the audio-only quality, or null
  /// if this stream has none.
  int? get _audioOnlyQualityIndex {
    for (var i = 0; i < _qualityObjects.length; i++) {
      final q = _qualityObjects[i];
      if (q.width == 0 && q.height == 0) return i + 1; // +1 for the Auto entry
    }
    return null;
  }
```

- [ ] **Step 3: Engage/disengage plugin background mode via a reaction**

In the constructor's `Platform.isAndroid` block (next to `_disposeAndroidAutoPipReaction`), add a reaction that mirrors the setting onto the plugin. The store uses individual `ReactionDisposer` fields (not a `reactions` list), so store the disposer:

```dart
      _disposeBackgroundPlaybackReaction = reaction(
        (_) => settingsStore.backgroundPlayback,
        (bool enabled) => _controller?.setBackgroundPlaybackEnabled(enabled),
        fireImmediately: true,
      );
```

- [ ] **Step 4: Subscribe to screen-state and apply the policy**

Where the controller is created/initialized (`_createController` / after `_controller` is set), subscribe:

```dart
  void _listenToScreenState() {
    _screenStateSub?.cancel();
    _screenStateSub = _controller?.screenStateStream.listen((screenOn) {
      if (!settingsStore.backgroundPlayback) return;
      runInAction(() {
        if (!screenOn) {
          final target = _screenLockAudioPolicy.onScreenOff(
            backgroundPlaybackEnabled: settingsStore.backgroundPlayback,
            playing: !_paused,
            userPaused: _userPaused,
            currentQualityIndex: _streamQualityIndex,
            audioOnlyQualityIndex: _audioOnlyQualityIndex,
            offline: _streamInfo == null,
            adBreakActive: _isAdBreakActive,
          );
          if (target != null) _setStreamQualityIndex(target);
        } else {
          final restore = _screenLockAudioPolicy.onScreenOn();
          if (restore != null) _setStreamQualityIndex(restore);
        }
      });
    });
  }
```

Call `_listenToScreenState()` right after the controller is created in `_createController`/init, and re-call it when the controller is recreated.

- [ ] **Step 5: Clean up**

The screen-state subscription is tied to the controller, so cancel it wherever the
controller is torn down (`_disposeController()`) — it's re-created by
`_listenToScreenState()` on the next controller. The reaction and policy live for the
store's lifetime, so dispose those only in `dispose()`.

In `_disposeController()`:

```dart
    _screenStateSub?.cancel();
    _screenStateSub = null;
    _screenLockAudioPolicy.reset();
```

In `dispose()`:

```dart
    _disposeBackgroundPlaybackReaction?.call();
    _disposeBackgroundPlaybackReaction = null;
```

- [ ] **Step 6: Add the settings toggle**

In `lib/screens/settings/video_settings.dart`, inside the `if (settingsStore.showVideo)` block, immediately after the "Native player (experimental)" switch, gated on the native player being on:

```dart
          if (settingsStore.showVideo && settingsStore.useNativePlayer)
            SettingsListSwitch(
              title: 'Background audio when screen locked',
              subtitle: const Text(
                'Keep the stream playing as audio-only when the screen is off. '
                'Native player only.',
              ),
              value: settingsStore.backgroundPlayback,
              onChanged: (newValue) =>
                  settingsStore.backgroundPlayback = newValue,
            ),
```

- [ ] **Step 7: Analyze**

Run: `flutter analyze lib/screens/channel/video/native_video_store.dart lib/screens/settings/video_settings.dart`
Expected: No issues (once the plugin branch exposing `setBackgroundPlaybackEnabled`/`screenStateStream` is referenced in `pubspec.yaml`).

- [ ] **Step 8: Run the full suite**

Run: `flutter test`
Expected: all pass (Tasks 1–2 tests included).

- [ ] **Step 9: Commit**

```bash
git add lib/screens/channel/video/native_video_store.dart lib/screens/settings/video_settings.dart
git commit -m "feat(video): switch to audio-only on screen lock when background playback is on"
```

---

## Verification (on-device, Android)

1. Enable native player + the new "Background audio when screen locked" toggle.
2. Play a live stream, lock the screen → audio continues indefinitely (not 1–5s), and quality drops to audio-only.
3. Unlock → prior video quality restored, video resumes.
4. Repeat from PiP (lock while in PiP) → audio continues.
5. Toggle off → audio stops on lock as before (no regression).
6. A17 probe: `adb shell dumpsys audio | grep AudioHardening` shows a FGS with while-in-use (not `level: partial`).

> Steps 2–6 require the plugin PR branch (foreground service + screen-state events) referenced in `pubspec.yaml`. Until then, Tasks 1–2 (setting + policy) are independently verified by their unit tests.
