# Controls (Control Center, Lock Screen, Action button)

Controls (iOS 18+) are a separate widget kind: small toggles and buttons the user places in **Control Center**, on the **Lock Screen**, or binds to the **Action button**. They are defined with `ControlWidget` (a sibling of `Widget`) and run `AppIntent`s, reusing the interactive-widget model. On macOS and watchOS, controls arrived in the **26 releases**, not 2024 - gate accordingly.

> Create a control via the Widget Extension template (File ▸ New ▸ Target ▸ Widget Extension ▸ **Include Control**), and add it to your `WidgetBundle`. The control's body runs in the widget-extension process. The `Control*` protocols are in **SwiftUI**; the configuration/template types are in **WidgetKit**.

## Anatomy

```swift
import WidgetKit
import SwiftUI
import AppIntents

struct StudyControl: ControlWidget {
    static let kind = "com.example.StudyControl"

    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: Self.kind) {
            ControlWidgetToggle(
                "Study Swift",
                isOn: Persistence.currentDay?.didStudy ?? false,
                action: SetStudiedIntent()
            ) { isOn in
                Label(isOn ? "Studied" : "Study", systemImage: isOn ? "checkmark.circle" : "swift")
            }
            .tint(.orange)
        }
        .displayName("Study Swift Today")
        .description("Mark that you studied Swift today.")
    }
}
```

A `ControlWidget` returns a `ControlWidgetConfiguration` whose content is **one template** (`ControlWidgetToggle` or `ControlWidgetButton`).

## When controls reload

The control body re-runs on three events: (1) its intent's `perform()` returns, (2) the app calls `ControlCenter.shared.reloadControls(ofKind:)`, (3) a push notification invalidates it (the system reloads push-updated controls for you). You never reload in response to push yourself.

## Toggle vs button - and the intent types

This is the most common mix-up:

- **`ControlWidgetToggle`** (binary on/off) takes a **`SetValueIntent`** whose `value: Bool` is *set by the system* to the new desired state - never set it yourself:

  ```swift
  struct SetStudiedIntent: SetValueIntent {
      static let title: LocalizedStringResource = "Set Studied"
      @Parameter(title: "Studied") var value: Bool      // the system writes the new state here

      func perform() async throws -> some IntentResult {
          let context = ModelContext(Persistence.container)
          // ... fetch today, set day.didStudy = value, save ...
          WidgetCenter.shared.reloadTimelines(ofKind: "StudyWidget")
          return .result()      // finish ALL work before returning - the reload follows
      }
  }
  ```

  There's also a `valueLabel:` form that switches the label by state:

  ```swift
  ControlWidgetToggle("Productivity Timer", isOn: value, action: ToggleTimerIntent()) { isOn in
      Label(isOn ? "Running" : "Stopped", systemImage: "timer")
  }
  ```

- **`ControlWidgetButton`** (one-shot action) takes a plain **`AppIntent`** (or an **`OpenIntent`** to launch the app):

  ```swift
  ControlWidgetButton(action: StartRecordingIntent()) {
      Label("Record", systemImage: "record.circle")
  }
  ```

Mixing them up (`SetValueIntent` on a button, plain `AppIntent` on a toggle) won't behave. **Toggle ⇒ `SetValueIntent`, button ⇒ `AppIntent`/`OpenIntent`.**

### Launching the app from a control

Use an `OpenIntent`. Its type must be a member of **both** the app target and the widget-extension target, or the launch fails:

```swift
struct LaunchAppIntent: OpenIntent {
    static let title: LocalizedStringResource = "Launch App"
    @Parameter(title: "Target") var target: AppScreen   // an AppEnum of destinations
}
```

## Dynamic state with a value provider

A control often reflects live state (is recording? is the light on?). Use a value provider:

```swift
struct RecordingControlValueProvider: AppIntentControlValueProvider {
    func previewValue(configuration: RecordingControlConfiguration) -> Bool { false }   // OFF state, sync, fast

    func currentValue(configuration: RecordingControlConfiguration) async throws -> Bool {
        AppGroupCoordinator.shared.isRecording      // async live read from the App Group
    }
}

struct RecordingControl: ControlWidget {
    var body: some ControlWidgetConfiguration {
        AppIntentControlConfiguration(
            kind: "RecordingControl",
            provider: RecordingControlValueProvider()
        ) { isRecording in
            ControlWidgetToggle("Recording", isOn: isRecording, action: ToggleRecordingIntent()) { isOn in
                Label("Toggle Recording", systemImage: "microphone.circle")
            }
        }
        .displayName("Toggle Recording")
        .description("Start or stop recording.")
    }
}
```

- `previewValue` is shown in the controls gallery; it must be **synchronous** and should be the **off/default** value.
- `currentValue` is the async live read; it runs in the widget/control extension, so read from the **App Group**.
- For a **non-configurable** control, use the provider form of `StaticControlConfiguration(kind:provider:)` with a `ControlValueProvider` (sync `previewValue` + async `currentValue()` with no configuration argument).
- For a **configurable** control, the configuration type is a `ControlConfigurationIntent` (its `@Parameter`s power the editing UI); `AppIntentControlValueProvider` receives it.

## Refinement modifiers

| Modifier | Lives on | Effect |
|---|---|---|
| `.displayName(_:)` | configuration | Per-control name in the gallery (defaults to the app name). |
| `.description(_:)` | configuration | Localized description in the configuration view. |
| `.promptsForUserConfiguration()` | configuration | Auto-prompt for configuration when the control is added, if it's useless unconfigured. |
| `.tint(_:)` | template / Label | Characteristic color of the symbol + value text in the ON state (default `systemBlue`). |
| `.controlWidgetActionHint(_:)` | the `Label` (View) | Custom Action-button hint; **start with a verb**. |
| `.controlWidgetStatus(_:)` | View | Momentary status text shown in Control Center when the action fires. Use sparingly. There is **no** `controlWidgetStatusText`. |
| `.privacySensitive(_:)` | template | Redacts state/label while the device is unauthenticated. |
| `.disabled(_:)` | View | Standard SwiftUI disable. |
| `authenticationPolicy` | the `AppIntent` | `.requiresAuthentication` gates `perform()` behind device auth. |

Action-button hint defaults: button → `"Hold for 'Perform Action'"`; button+`OpenIntent` → `"Hold to Open MyApp"`; toggle → `"Hold to Turn On/Off 'Name'"`; a `controlWidgetActionHint` overrides these.

```swift
ControlWidgetToggle("Light", isOn: isOn, action: ToggleLightIntent()) { isOn in
    Label(isOn ? "On" : "Off", systemImage: "lightbulb")
}
.controlWidgetStatus(isOn ? "Light is on" : "Light is off")
```

## Reloading controls

After state changes (from the app or another intent), refresh with `ControlCenter`, the parallel of `WidgetCenter`:

```swift
ControlCenter.shared.reloadControls(ofKind: "StudyControl")
ControlCenter.shared.reloadAllControls()
```

A toggle intent that flips widget *and* control state should reload both centers. During development, enable WidgetKit developer mode to strip the reload budget.

## Platform availability

- Controls: **iOS 18+**, **macOS 26+** (Control Center + menu bar), **watchOS 26+** (Control Center, Action button, Smart Stack). `ControlCenter` is iOS 18 / macOS 26 / watchOS 26.
- `SetValueIntent` exists since iOS 18 / macOS 15 (it predates control support on the Mac).
- On watchOS, controls and Live Activities from a paired iPhone are surfaced automatically; you don't rebuild them. A **native watch control** must be gated to watchOS 26 - a button control that opens the watch app to record (from VivaDicta):

  ```swift
  @available(watchOS 26, *)
  struct WatchRecordControl: ControlWidget {
      static let kind = "com.you.app.watchkitapp.RecordControl"
      var body: some ControlWidgetConfiguration {
          StaticControlConfiguration(kind: Self.kind) {
              ControlWidgetButton(action: OpenRecorderIntent()) {   // openAppWhenRun = true
                  Label("Quick Record", systemImage: "mic.fill")
              }
          }
          .displayName("Quick Record")
          .description("Open the app and record a voice note.")
      }
  }
  ```

## Checklist

- `ControlWidgetToggle` ⇒ `SetValueIntent` (`value: Bool` set by system, finish work before returning); `ControlWidgetButton` ⇒ `AppIntent`/`OpenIntent` (the `OpenIntent` type must be in both targets).
- Live state via a value provider: sync `previewValue` (off state), async `currentValue` (read App Group).
- Reload with `ControlCenter.shared.reloadControls(ofKind:)`; mark helper intents `isDiscoverable = false`.
- Status modifier is `controlWidgetStatus(_:)`, not `controlWidgetStatusText`; hints start with a verb; `promptsForUserConfiguration()` for configure-on-add.
- Controls are iOS 18 but **macOS/watchOS 26** - gate availability.
