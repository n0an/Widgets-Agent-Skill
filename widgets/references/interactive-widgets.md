# Interactive widgets

Since iOS 17 a widget can contain a `Button` or `Toggle` that runs an `AppIntent` **without opening the app**. This is the *only* interactivity widgets have - there are no gestures, no `onTapGesture`, no closures. Understanding the process boundary is everything.

## The two interactive controls

```swift
import AppIntents
import WidgetKit

// A button that runs an action:
Button(intent: AddDrinkIntent(amount: 1)) {
    Label("Add", systemImage: "plus.circle.fill")
}

// A toggle bound to a Bool, flipping it via an intent:
Toggle(isOn: entry.isFavorite, intent: ToggleFavoriteIntent(id: entry.id)) {
    Text("Favorite")
}
```

- Both take an `AppIntent`. Only `Button(intent:)` and `Toggle(intent:)` are interactive - **other controls won't work** in a widget. A `Link` / `widgetURL` opens the app; everything else is inert.
- Both require `import SwiftUI` + `import AppIntents`.

## The process boundary (the part people get wrong)

The widget runs as a **separate process**, and its view code runs only while the system *archives* the view - it is not running while the widget is on screen. So when the user taps the button:

1. User taps `Button(intent:)` in the widget.
2. The system executes the intent's `perform()` (in the **app's** process - the app is launched in the background if needed - so it can write the real, writable data store).
3. `perform()` mutates the shared data (App Group store / SwiftData / Core Data).
4. On return, **the system immediately and reliably reloads the widget's timeline.** This interaction-triggered reload is *guaranteed* (ordinary scheduled reloads are best-effort).
5. The widget extension regenerates its timeline and re-renders, animating the diff (see `animations-and-transitions.md`).

```swift
struct AddDrinkIntent: AppIntent {
    static let title: LocalizedStringResource = "Add Drink"
    static let isDiscoverable = false           // helper intent: keep it out of Shortcuts

    @Parameter(title: "Amount") var amount: Int  // ONLY @Parameter state is persisted to perform()
    init() {}
    init(amount: Int) { self.amount = amount }

    func perform() async throws -> some IntentResult {
        let store = DrinkStore(suiteName: "group.com.you.app")   // shared container
        store.add(amount)                                        // persist BEFORE returning
        WidgetCenter.shared.reloadTimelines(ofKind: "CaffeineWidget")
        return .result()
    }
}
```

Rules that follow from this:

- **Any state the intent needs at perform-time must be a `@Parameter`.** Plain stored properties are not persisted across the archive boundary into `perform()`.
- **Write to the App Group**, not `UserDefaults.standard`. The widget reads the same container.
- **Commit before returning.** Persist the change (`save()`) inside `perform()` before it returns, because the guaranteed reload fires on return.
- **Mark widget-only intents `static let isDiscoverable = false`** so they don't appear in Shortcuts/Spotlight as user-runnable actions.
- **Keep `perform()` fast** - it runs to update a glanceable view.

### `Toggle` is optimistic - and your `ToggleStyle` must cooperate

A `Toggle(isOn:intent:)` flips its presentation **immediately**, without waiting for the round-trip, because the system pre-renders **both** `isOn` states at archive time. For that to work, a **custom `ToggleStyle` must read `configuration.isOn`** to choose its appearance - if it ignores `configuration.isOn` (e.g. keys off your own model), the optimistic flip can't render and the toggle feels stuck:

```swift
struct CheckToggleStyle: ToggleStyle {
    func makeBody(configuration: Configuration) -> some View {
        Label {
            configuration.label
        } icon: {
            Image(systemName: configuration.isOn ? "checkmark.circle.fill" : "circle")  // MUST read isOn
        }
    }
}
```

### SwiftData / Core Data from an intent

`@Query` only works in a `View`; from an intent run a one-shot fetch on a fresh context:

```swift
func perform() async throws -> some IntentResult {
    let context = ModelContext(Persistence.container)   // container URL in the App Group
    let today = Calendar.current.startOfDay(for: .now)
    let descriptor = FetchDescriptor<Day>(predicate: #Predicate { $0.date == today })
    guard let day = try context.fetch(descriptor).first else { return .result() }
    day.didStudy.toggle()
    try context.save()
    WidgetCenter.shared.reloadTimelines(ofKind: "StudyWidget")
    return .result()
}
```

See `data-and-networking.md` for the shared-store setup.

## Opening the app from an intent, and fire-and-forget signals

Two related needs that aren't the usual "mutate the store + reload":

- **Launch the app as part of the action.** Set `static var openAppWhenRun: Bool { true }` on the intent (or use an `OpenIntent`); the system foregrounds the app when `perform()` runs. Useful for a Record/Compose button that must bring the app forward to do its work.

  ```swift
  struct OpenRecorderIntent: AppIntent {
      static var title: LocalizedStringResource { "Record Voice Note" }
      static var openAppWhenRun: Bool { true }          // foreground the app
      func perform() async throws -> some IntentResult {
          RecordingSignal.fire()                         // tell the running app to start
          return .result()
      }
  }
  ```

- **Signal the running app without sharing writable state.** When you only need to *tell* the app "do this now" (start recording, refresh) rather than hand it data, a Darwin notification is a lightweight cross-process ping - no App Group write required:

  ```swift
  CFNotificationCenterPostNotification(
      CFNotificationCenterGetDarwinNotifyCenter(),
      CFNotificationName("com.you.app.toggleRecording" as CFString),
      nil, nil, true)
  // The app observes the same Darwin name and reacts.
  ```

  Use the App Group for *data* the widget and app both read/write; use a Darwin notification for a *fire-and-forget signal*. (This is how VivaDicta's watch Record control wakes the app to start recording.)

## Optimistic UI: `invalidatableContent`

There's a brief gap between the tap and the reloaded timeline (most visible for an iPhone widget on a Mac). Mark content that's about to change so the system shows a "pending" treatment until the next entry arrives:

```swift
Text("\(entry.count)")
    .invalidatableContent()      // system renders it as pending (RedactionReasons.invalidated)
```

You can also read `@Environment(\.redactionReasons)` to style the pending state. Use it sparingly, only on values that actually change. See `animations-and-transitions.md`.

## The 27-releases gotcha: write intents must run on `.main`

In the 27 releases, the **widget extension's view of the shared store is read-only**, and a widget button whose intent *writes* will hit a limit if it runs in the extension. Pin write intents to the app process and display-only intents to the widget extension:

```swift
struct UpdateFavoriteIntent: AppIntent {              // writes → must be the main app
    static var allowedExecutionTargets: ExecutionTargets { .main }
}

struct GetStatusIntent: AppIntent {                   // read-only → widget extension is fine
    static var allowedExecutionTargets: ExecutionTargets { .widgetKitExtension }
}
```

Values are `.main`, `.appIntentsExtension`, `.widgetKitExtension`, or an array.

## Live Activity buttons

Inside a Live Activity, use `LiveActivityIntent` (allowed to run from the Lock Screen / Dynamic Island) instead of a plain `AppIntent`:

```swift
struct EndSessionIntent: LiveActivityIntent {
    static let title: LocalizedStringResource = "End Session"
    static let isDiscoverable = false
    @Parameter(title: "Active") var isActive: Bool
    init() { isActive = true }
    init(isActive: Bool) { self.isActive = isActive }

    @MainActor func perform() async throws -> some IntentResult {
        SessionCoordinator.shared.endFromLiveActivity()
        return .result()
    }
}

Button(intent: EndSessionIntent(isActive: false)) {
    Image(systemName: "power.circle.fill")
}
.buttonStyle(.plain)
```

In a **Control**, a toggle's action is a `SetValueIntent`, not a plain `AppIntent` - see `controls.md`.

## Checklist

- Interactivity = `Button(intent:)` / `Toggle(intent:)` only; gestures/closures do nothing.
- The intent runs in the **app process**; state it needs must be `@Parameter`; write to the **App Group**, commit, then **reload** (`WidgetCenter`) - the post-tap reload is guaranteed.
- Custom `ToggleStyle` must read `configuration.isOn` for the optimistic flip to render.
- Mark widget-only intents `isDiscoverable = false`; keep `perform()` fast; `invalidatableContent()` for pending state.
- 27 releases: write intents → `allowedExecutionTargets = .main`; read-only → `.widgetKitExtension`.
- Live Activity buttons use `LiveActivityIntent`; Control toggles use `SetValueIntent`.
