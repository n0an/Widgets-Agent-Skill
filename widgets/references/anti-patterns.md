# Anti-patterns: common widget mistakes

WidgetKit has grown across many releases - interactive widgets (17), Controls (18), the tinted Home Screen and push-updated widgets (26), the landscape Dynamic Island (27) - and most training data predates the modern shape. These are the mistakes to catch, each with a wrong/right pair or a concrete fix.

## Process model

### Reading `UserDefaults.standard` in the extension
The widget runs in a separate process; `UserDefaults.standard` is a different sandbox. Use the App Group.
```swift
// WRONG
let count = UserDefaults.standard.integer(forKey: "count")
// RIGHT
let count = UserDefaults(suiteName: "group.com.you.app")!.integer(forKey: "count")
```

### Sharing an in-memory store / singleton between app and widget
They don't share memory. An `@Observable` model or static cache in the app is invisible to the extension. Persist to the App Group container (UserDefaults suite, file, or a shared `ModelContainer`/Core Data URL) and read it in the provider.

### Expecting the widget view to be "live"
The system **archives the view and renders it later**. Your code is not running on screen. Anything that depends on continuous execution (timers you tick, `onReceive` of a Combine publisher, `Task` loops) does nothing. Use timeline entries and `Text(_:style:)`.

## Interactivity

### Using gestures / closures for taps
```swift
// WRONG - silently inert in a widget
Image(systemName: "plus").onTapGesture { add() }
// RIGHT
Button(intent: AddIntent()) { Image(systemName: "plus") }   // iOS 17+
```

### Mutating data in the intent but never reloading
The widget won't refresh itself. After the write, reload:
```swift
try context.save()
WidgetCenter.shared.reloadTimelines(ofKind: "MyWidget")   // exact kind string
```

### Writing from the widget extension's process
An interactive `AppIntent.perform()` runs in the **app** process for a reason - it can write the shared store. Don't try to write from a value provider or timeline (extension). In the 27 releases, pin write intents with `static var allowedExecutionTargets: ExecutionTargets { .main }`.

### Helper intents polluting Shortcuts
A button-backing intent shows up as a user action unless you opt out:
```swift
static let isDiscoverable = false
```

### State the intent needs isn't a `@Parameter`
Only `@Parameter` properties survive the archive boundary into `perform()`. A plain stored property is lost. Anything the intent must know at perform-time (an id, an amount, a target state) must be a `@Parameter`.

### Custom `ToggleStyle` that ignores `configuration.isOn`
A widget `Toggle` flips optimistically because the system pre-renders both states. A custom `ToggleStyle` must switch on `configuration.isOn`; if it keys off your own model instead, the optimistic flip can't render and the toggle feels stuck.

## Timeline & reloads

### Polling with a tight reload policy
```swift
// WRONG - exhausts the daily budget by mid-morning, then freezes
Timeline(entries: [Entry(date: .now, ...)], policy: .after(.now.addingTimeInterval(60)))
// RIGHT - many pre-computed entries cover the window; reload at the end
Timeline(entries: hourlyEntries, policy: .atEnd)
```

### Reloading to animate
You don't reload to move a clock or countdown - use self-updating text:
```swift
Text(endDate, style: .timer)        // one entry covers the whole countdown
Text(date, style: .relative)
```

### Blocking work in `placeholder`
`placeholder(in:)` must return synchronously with stand-in data. Move fetching to `getSnapshot`/`getTimeline`.

### Empty gallery snapshot
`getSnapshot` feeds the gallery; return realistic, populated data, not an empty/loading state, or users won't add the widget.

### `.after(_:)` with a `TimeInterval`
`TimelineReloadPolicy` is a **struct** and `.after(_:)` takes a **`Date`**:
```swift
.after(Date.now.addingTimeInterval(3600))   // not .after(3600)
```

## Backgrounds & rendering modes

### Missing `containerBackground`
Without `.containerBackground(for: .widget)` the widget is blank in the gallery and can't render the material/tint substitution. Always include it (iOS 17+). Note the type is `ContainerBackgroundPlacement` (`.widget`) - there is no `WidgetBackgroundPlacement`.

### Opaque accessory background where you wanted transparency
Omitting `containerBackground` makes the system add an **opaque** default. For a see-through Lock Screen / complication background, give it an **empty** builder:
```swift
.containerBackground(for: .widget) { }
```

### Treating rendering modes as enums
`WidgetRenderingMode` and `WidgetAccentedRenderingMode` are **structs with type properties**, not enums. Compare with `==`:
```swift
if renderingMode == .accented { ... }
```

### Ignoring accented / tinted mode
Designing only for `.fullColor` yields white blobs and invisible content on the Lock Screen / tinted Home Screen. Read `@Environment(\.widgetRenderingMode)`, put primary content in the accent group with `.widgetAccentable()`, and for images use `widgetAccentedRenderingMode(.fullColor / .desaturated / .accentedDesaturated)`. Tinting keys off **alpha** - a multicolor opaque bitmap becomes a flat block; run it through `.luminanceToAlpha()` first to get a clean tintable glyph.

### Assuming `widgetAccentedRenderingMode` is iOS 26
It's **iOS 18**. WWDC 2025 re-spotlighted it because the tinted/clear Home Screen made it ubiquitous, but gate it at iOS 18, not 26.

## Configuration

### `perform()` on a `WidgetConfigurationIntent`
A configuration intent needs **no** `perform()`; it's a pure parameter carrier in the AppIntents module.

### Migrating a pre-17 configurable widget without `CustomIntentMigratedAppIntent`
Existing user configurations break. Conform the new App Intent to `WidgetConfigurationIntent` **and** `CustomIntentMigratedAppIntent`, keeping the old `intentClassName`. (Only needed for migration; brand-new configurable widgets just adopt `WidgetConfigurationIntent`.)

### No default on configuration parameters
The widget shows nothing until configured. Give every `@Parameter` a sensible default; prefer multiple widget instances over a multi-select parameter.

## Controls

### Wrong intent type for the template
Toggle ⇒ `SetValueIntent` (its `value: Bool` is set by the system). Button ⇒ `AppIntent` (or `OpenIntent` to launch). Swapping them won't behave.

### `controlWidgetStatusText`
There is no such modifier. The status modifier is `controlWidgetStatus(_:)`.

### Assuming controls are on macOS/watchOS in 2024
Controls are iOS 18, but **macOS 26 / watchOS 26**. Gate accordingly.

### Forgetting to reload controls
Use `ControlCenter.shared.reloadControls(ofKind:)` (the parallel of `WidgetCenter`) after state changes.

## Live Activities

### Putting changing data in `ActivityAttributes`
Only the nested `ContentState` updates. Static identity goes in `ActivityAttributes`; everything that changes goes in `ContentState` (Codable, Hashable, < 4 KB).

### `staleDate` / `relevanceScore` on the configuration
They live on **`ActivityContent.init(state:staleDate:relevanceScore:)`**, not the `ActivityConfiguration`.

### Expecting `Activity.startToken` / a `pushToStart` `PushType`
Push-to-start uses `Activity.pushToStartToken` / `pushToStartTokenUpdates` (iOS 17.2+) and an APNs payload with `event: "start"`. `PushType` only has `.token` and `.channel(_:)`.

### Missing Info.plist keys
`NSSupportsLiveActivities` = YES in the **app** target; add `NSSupportsLiveActivitiesFrequentUpdates` for high-frequency pushes.

### `symbolEffect` in a Live Activity or widget
Not supported - it silently no-ops. Use static SF Symbols + `Text` timers.

### Reading the push token synchronously
`activity.pushToken` is usually `nil` right after `request(...)` and the token rotates over the activity's life. Read it from the **async sequence** `activity.pushTokenUpdates` and re-send on every change.

### Encoding `content-state` with a custom JSONEncoder strategy
The device decodes a push's `content-state` with a default `JSONDecoder`. If your server encodes with custom key/date strategies, decoding fails silently and the activity won't update. Use a plain `JSONEncoder`.

### Wrong APNs topic / push type for a Live Activity
Headers must be `apns-push-type: liveactivity` and `apns-topic: <bundleID>.push-type.liveactivity`. A normal `alert` push type won't update the activity.

## Data & fetching

### `AsyncImage` (or any fetch) inside the widget view
The view is archived and rendered later, so a fetch at render time won't run reliably. Download and **downsample** images during timeline generation and put them in the entry.

### A provider that never completes
On a network error, still call the completion (or return) with last-good or placeholder data. A provider that hangs wastes the reload budget and freezes the widget.

### `@Query` in a provider or intent
`@Query` only works in a SwiftUI `View`. In providers/intents use a one-shot `FetchDescriptor` on a `ModelContext` whose store is in the App Group.

## Migration / legacy

### Using `IntentConfiguration` / `.intentdefinition` for a new configurable widget
That's the pre-iOS-17 SiriKit path. New widgets use `AppIntentConfiguration` + `WidgetConfigurationIntent` + `AppIntentTimelineProvider`. Only keep `IntentConfiguration` behind `#available` if you deploy below iOS 17.

### Migrating a configurable widget and losing user configs
Conform the new App Intent to `CustomIntentMigratedAppIntent`, set `intentClassName` to the old SiriKit class name, and keep every parameter's name and type identical, or the stored config is dropped.

### Building a new watch complication with ClockKit
ClockKit is migration-only. Build complications as WidgetKit accessory widgets; provide a `CLKComplicationWidgetMigrator` for existing faces.

### No `recommendations()` on a configurable watchOS widget
The watch has no configuration editor, so a configurable widget must return preconfigured `AppIntentRecommendation`s from `recommendations()`, or it can't be built/added.

### Not handling the landscape Dynamic Island (27 releases)
Compact/minimal views now render in landscape with constrained width. Read `@Environment(\.isDynamicIslandLimitedInWidth)` and fall back to a narrower view, or content clips.

## Platform / availability

### Calling the WWDC 2026 cohort "iOS 19"
It's **iOS 27** - "the 27 releases". WWDC 2025 = "the 26 releases".

### Listing `.accessoryCorner` for an iOS widget
`.accessoryCorner` is **watchOS only**. And accessory families are **iOS 16+**, not 17.

### Shipping a family you didn't lay out
`supportedFamilies` should list only families you actually designed; switch on `\.widgetFamily` with a `default` branch so a new family doesn't render empty.

## Quick API-name corrections

| Wrong / assumed | Correct |
|---|---|
| `WidgetBackgroundPlacement` | `ContainerBackgroundPlacement` (value `.widget`) |
| `WidgetRenderingMode` / `WidgetAccentedRenderingMode` are enums | they're **structs** with type properties |
| `.after(timeInterval)` | `.after(date)` - takes a `Date`; policy is a struct |
| `controlWidgetStatusText` | `controlWidgetStatus(_:)` |
| `widgetAccentedRenderingMode` is iOS 26 | iOS **18** |
| `Activity.startToken` / `.pushToStart` PushType | `pushToStartToken` / `pushToStartTokenUpdates` (separate mechanism) |
| `ActivityFamily.large` | only `.small` and `.medium` exist |
| Controls on macOS/watchOS in iOS 18 cycle | controls are macOS/watchOS **26** |
