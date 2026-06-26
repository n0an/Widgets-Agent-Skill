# Fundamentals: the widget execution model

A widget is a small, glanceable view of your app's content that the system renders on the Home Screen, Lock Screen, StandBy, the desktop, the Apple Watch, and in the Smart Stack. The single most important fact about widgets shapes everything else:

> A widget is **not your app**. It runs in a **separate extension process**, it is **always SwiftUI**, and the system **archives your view and renders the snapshot later** - your code is not running while the widget is on screen.

This file is the mental model. Read it first; the API-specific files build on it.

## The process boundary, and the App Group

Your app and your widget extension are two separate processes with two separate sandboxes. They share nothing by default. Three consequences:

- **They cannot see each other's memory.** An in-memory singleton, an `@Observable` store, or `UserDefaults.standard` in the app is invisible to the widget. Share state only through an **App Group**: `UserDefaults(suiteName: "group.com.you.app")`, a file in the group container, or a `ModelContainer` / Core Data store whose URL is in the group container.
- **The widget extension has limited resources and time.** Timeline generation runs under tight memory and wall-clock limits; heavy work gets the extension killed. Fetch the minimum, cache aggressively in the shared container, and let the app do the expensive work.
- **In the 27 releases the widget extension's view of the shared store is read-only.** Interactive intents that *write* must run in the app process (see `interactive-widgets.md`).

Add the App Group capability to **both** targets, with the same group identifier. A widget that reads `UserDefaults.standard` instead of the suite is the most common "my widget shows nothing / shows stale data" bug.

### Setting up the App Group

1. In Signing & Capabilities, add **App Groups** to the **app** target and the **widget extension** target, and enable the **same** group id (`group.com.you.app`) on both.
2. Share small values via the suite, larger data via the container directory:

   ```swift
   // Small codable/scalar state:
   let defaults = UserDefaults(suiteName: "group.com.you.app")!

   // Files / a SwiftData or Core Data store URL:
   let container = FileManager.default
       .containerURL(forSecurityApplicationGroupIdentifier: "group.com.you.app")!
   ```

3. Keep the group id (and each widget's `kind`) in a **shared constants** file compiled into both targets, so the app and extension can't drift on a string. See `data-and-networking.md` for the read/write patterns.

## The widget bundle and the `Widget` protocol

A widget extension's entry point is a `WidgetBundle`. Each widget is a type conforming to `Widget`, returning a `WidgetConfiguration` from its `body`. One extension can vend many widgets (and controls and Live Activities) from one bundle:

```swift
@main
struct MyWidgetBundle: WidgetBundle {
    var body: some Widget {
        CaffeineWidget()          // Home Screen
        CaffeineLockWidget()      // Lock Screen / accessory
        CaffeineControl()         // Control Center (ControlWidget)
        CaffeineLiveActivity()    // Live Activity (ActivityConfiguration)
    }
}

struct CaffeineWidget: Widget {
    let kind = "CaffeineWidget"   // stable identifier; reused by WidgetCenter reloads

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            CaffeineWidgetView(entry: entry)
                .containerBackground(for: .widget) { BackgroundGradient() }
        }
        .configurationDisplayName("Caffeine")     // shown in the gallery
        .description("Track today's caffeine.")    // shown in the gallery
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

- `kind` is a stable string that identifies the widget for the lifetime of the app. `WidgetCenter.shared.reloadTimelines(ofKind:)` keys off it. Keep `kind` strings in a shared constant so the app and extension cannot drift.
- `configurationDisplayName` / `description` are the gallery copy.
- `supportedFamilies` declares exactly which sizes you laid out. See `families-and-layout.md`.

The three building blocks of a configuration are the **kind**, the **provider** (the timeline - see `timeline-provider.md`), and the **view** (a SwiftUI view that takes one entry). Everything else is a modifier on the configuration or the view.

## The timeline lifecycle (the heart of it)

The system asks your provider for a **timeline**: a sequence of dated `TimelineEntry` snapshots plus a policy for when to ask again. The system renders the entry whose `date` is current, swaps to the next entry as time passes, and re-requests the timeline according to the policy or when you call `WidgetCenter`. Three provider callbacks:

1. **`placeholder(in:)`** - a synchronous, instant stand-in used for first render and as the redacted skeleton. No fetching.
2. **`getSnapshot` / `snapshot(for:in:)`** - a single entry for the **widget gallery** and transient states. Return realistic, populated data even when the user has none.
3. **`getTimeline` / `timeline(for:in:)`** - the real sequence of entries plus a `TimelineReloadPolicy`.

See `timeline-provider.md` for the full contract, reload policies, and the reload budget. The key discipline: **timelines are budgeted and opportunistic, not timers.** You describe future states; the system decides when to render and refresh them.

## `containerBackground` - mandatory, not optional

Always mark the widget's background with `.containerBackground(for: .widget) { ... }` (iOS 17+):

```swift
CaffeineWidgetView(entry: entry)
    .containerBackground(for: .widget) {
        LinearGradient(colors: [.indigo, .purple], startPoint: .top, endPoint: .bottom)
    }
```

The system needs to know which part of your view is "background" so it can:

- show the widget correctly in the **gallery** and during add/remove/relocate,
- substitute a **material** background on the Lock Screen, in StandBy, and on the desktop,
- apply **tint** in the tinted/clear Home Screen and accessory contexts.

The placement type is `ContainerBackgroundPlacement` and the value is `.widget` - there is no `WidgetBackgroundPlacement`. For an accessory widget or watch complication that should let the wallpaper/face show through, pass an **empty** builder: `.containerBackground(for: .widget) { }`. Omitting `containerBackground` entirely is worse than an empty one - the system adds an opaque default material. See `lock-screen-and-watch.md`.

Use `.contentMarginsDisabled()` on the configuration when you want to draw edge-to-edge (full-bleed gradient, image) and manage your own padding; otherwise the system applies default content margins so content isn't clipped at the corners.

## Deep linking back into the app

Two mechanisms to open the app from a tap:

- **`.widgetURL(_:)`** - one URL for the whole widget (or the current family's view). The app receives it via `.onOpenURL` / scene delegate.
- **`Link(destination:)`** - per-region links inside `systemMedium`/`systemLarge`/`systemExtraLarge`. Multiple tappable regions, each with its own URL. (`Link` is ignored on accessory/Lock Screen families, which get a single `widgetURL`.)

```swift
// Whole-widget tap target:
.widgetURL(URL(string: "myapp://record"))

// Per-region in a medium/large widget:
Link(destination: URL(string: "myapp://streak")!) { StreakView() }
Link(destination: URL(string: "myapp://calendar")!) { CalendarGrid() }
```

For actions that should run *without* opening the app (toggle, increment, mark done), use an `AppIntent` button instead - see `interactive-widgets.md`.

## Which surface am I building?

| Surface | API | Read |
|---|---|---|
| Home Screen widget | `StaticConfiguration` / `AppIntentConfiguration` | `timeline-provider.md`, `families-and-layout.md` |
| Lock Screen / StandBy / watch accessory | accessory `WidgetFamily` | `lock-screen-and-watch.md` |
| Tinted / accented appearance | `widgetRenderingMode` & friends | `rendering-modes-and-tinting.md` |
| Tappable button / toggle in a widget | `Button(intent:)` / `Toggle(intent:)` | `interactive-widgets.md` |
| Control Center / Action button / Lock Screen control | `ControlWidget` | `controls.md` |
| Ongoing live event (delivery, game, timer) | `ActivityConfiguration` (ActivityKit) | `live-activities.md` |
| Smart Stack relevance / rotation | `relevance()` / `RelevanceConfiguration` | `relevance-and-smart-stacks.md` |
| visionOS / CarPlay / macOS specifics | spatial & platform modifiers | `platforms.md` |

## The non-negotiables

- Share data only through an **App Group**; never `UserDefaults.standard` or in-memory state.
- `placeholder` is **synchronous**; the gallery snapshot is **realistic, populated** data.
- Always set `.containerBackground(for: .widget)`.
- Reloads are **budgeted and opportunistic**; describe future entries, don't expect a timer.
- The widget view is a **static archived snapshot** rendered later - the only interactivity is `AppIntent` buttons/toggles and deep links; gestures and `symbolEffect` do nothing.
