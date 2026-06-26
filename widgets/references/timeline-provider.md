# Timeline providers, reload policies, and the reload budget

The provider supplies the entries the system renders and tells it when to ask again. Getting the provider and the reload policy right is what separates a widget that stays fresh from one that "stops updating".

## Why timelines (not live views)

A widget is **stateless** and is **not running while it's on screen**. WidgetKit asks your provider for a sequence of view states ahead of time, **serializes each rendered view to disk**, and just-in-time renders the entry whose date is current - no app or extension process runs to display it. This is deliberate: the Home Screen is viewed dozens of times a day for seconds at a time, and the system can't afford to launch every app to draw a tile. So instead of "give me the current value", the contract is "describe the next several states and when to ask again". Everything below follows from that.

## `TimelineProvider` (StaticConfiguration)

Completion-handler based. Three callbacks:

```swift
struct Provider: TimelineProvider {
    func placeholder(in context: Context) -> Entry {
        Entry(date: .now, value: .sample)            // synchronous, instant
    }

    func getSnapshot(in context: Context, completion: @escaping (Entry) -> Void) {
        completion(Entry(date: .now, value: .sample)) // gallery + transient state
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> Void) {
        let entries = makeEntries()
        completion(Timeline(entries: entries, policy: .atEnd))
    }
}

struct Entry: TimelineEntry {
    let date: Date          // required: when this entry becomes current
    let value: Value
}
```

- `placeholder(in:)` must return **synchronously** with stand-in data - no disk, no network. It's used for first paint and as the `.redacted(reason: .placeholder)` skeleton.
- `getSnapshot` is what the **gallery** renders and what the system uses for transient/quick-look. Return realistic, fully-populated data, never an empty/loading state. (In most widgets, the timeline's first entry and the snapshot are the same entry.)
- `getTimeline` returns the real `[Entry]` plus a policy.

### `TimelineProviderContext`

The `context` passed to each callback exposes:
- `context.family` - the `WidgetFamily` being requested, so you can size your fetch and layout.
- `context.isPreview` - `true` during gallery snapshotting; return fast (use sample data, skip network).
- `context.displaySize` - the point size for this family on this device.
- `context.environmentVariants` - the appearance variants (color schemes, etc.) the system may render.

## `AppIntentTimelineProvider` (AppIntentConfiguration)

For user-configurable widgets (iOS 17+). Async, no completion handlers, and **the configuration intent is the first argument**:

```swift
struct Provider: AppIntentTimelineProvider {
    func placeholder(in context: Context) -> Entry {
        Entry(date: .now, configuration: ConfigurationIntent())
    }

    func snapshot(for configuration: ConfigurationIntent, in context: Context) async -> Entry {
        Entry(date: .now, configuration: configuration)
    }

    func timeline(for configuration: ConfigurationIntent, in context: Context) async -> Timeline<Entry> {
        let entries = await makeEntries(for: configuration)
        return Timeline(entries: entries, policy: .after(.now.addingTimeInterval(3600)))
    }
}
```

The entry carries the configuration so the view can read the user's choices. This is the modern replacement for `IntentTimelineProvider` (SiriKit) - see `configuration-and-intents.md` and `migration-and-deprecations.md`.

### `recommendations()` for preconfigured widgets

On surfaces with no configuration editor (watchOS Smart Stack/complications, StandBy), the system can't show your configuration UI, so you supply a preconfigured list. Override `recommendations()`:

```swift
func recommendations() -> [AppIntentRecommendation<ConfigurationIntent>] {
    Account.all.map { account in
        AppIntentRecommendation(intent: ConfigurationIntent(account: account), description: account.name)
    }
}
```

- On **watchOS this is effectively required** to build a configurable widget. Return `[]` to let people configure freely where a UI exists.
- The legacy variant returns `[IntentRecommendation]`; the App Intents variant returns `[AppIntentRecommendation<Intent>]`.
- When the recommended set changes, call `WidgetCenter.shared.invalidateConfigurationRecommendations()`.

## The `Timeline` and reload policies

A `Timeline` is `[Entry]` plus a `TimelineReloadPolicy`. **`TimelineReloadPolicy` is a struct**, and `.after(_:)` takes a **`Date`** (not a `TimeInterval`):

- **`.atEnd`** - after the last entry's date passes, ask for a new timeline. Use when you precompute a window and can compute the next window when this one runs out.
- **`.after(date)`** - ask again no sooner than `date`. Use when you know the next interesting moment (a countdown hits zero, an event starts).
- **`.never`** - don't reload on a schedule. Use when content only changes from interaction, push, or an explicit `WidgetCenter` reload.

A common pattern - cover the next 24 hours with one entry per period, then reload at the end:

```swift
func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> Void) {
    let now = Date()
    let entries = (0..<24).compactMap { hour in
        Calendar.current.date(byAdding: .hour, value: hour, to: now)
            .map { Entry(date: $0, value: value(at: $0)) }
    }
    let reloadDate = Calendar.current.date(byAdding: .hour, value: 24, to: now)!
    completion(Timeline(entries: entries, policy: .after(reloadDate)))
}
```

Providing many entries lets the widget visibly change over time (a clock, a mesh-gradient that drifts, an hourly forecast) **without** spending reloads - the system swaps pre-supplied snapshots. Use this instead of reloading frequently.

### Time-based text doesn't need entries

For a live-updating clock or countdown, you don't need an entry per second. `Text` has self-updating styles the system re-renders:

```swift
Text(event.date, style: .relative)   // "in 3 minutes", updates live
Text(event.date, style: .timer)      // counts down/up
Text(event.date, style: .offset)     // "+3:14" offset from now
Text(timerInterval: start...end)     // range timer
Text(event.date, style: .date)
Text(event.date, style: .time)
ProgressView(timerInterval: start...end)   // auto-advancing bar
Gauge(value: level, in: 0...max) { Text("mg") }
```

One entry can show a running timer or progress bar for its whole duration. The string-interpolation form `"\(date, style: .relative) ago"` auto-generates a localized key.

## The reload budget (why widgets "stop updating")

The system imposes a **daily reload budget** per widget. The canonical guidance: a frequently-viewed widget receives roughly **40-70 background reloads per day** - about one every 15-30 minutes if evenly spaced - and the budget is scaled by how often the user actually looks, by battery, by Low Power Mode, and by foreground load. Implications:

- **You cannot poll.** A widget that requests `.after(60s)` forever exhausts its budget by mid-morning and then goes stale for the rest of the day.
- The system **learns viewing habits** and front-loads reloads when the user tends to look; it may withhold reloads overnight.
- During development, enable **WidgetKit developer mode** (Settings ▸ Developer) to lift the budget so you can iterate. Never rely on dev-mode frequency in production reasoning.

### Reloads that do NOT count against the budget

Some reloads are free, which is why a well-designed widget can stay fresh without polling:

- the **container app is in the foreground**, or in an active user session (audio Now Playing, navigation);
- a **significant location change** (for location-using widgets);
- a **presentation-environment change** - Dynamic Type size, bold text, language/region, accent/appearance, account change, a significant time change;
- a system-granted **staleness reload**.

Levers that don't spend budget at all: pre-supplied entries, self-updating `Text` timers, and `containerBackground` material that the system tints for free.

## Reloading on demand with `WidgetCenter`

From the **app** (e.g. after the user changes data) or from an **interactive intent**:

```swift
import WidgetKit

WidgetCenter.shared.reloadAllTimelines()
WidgetCenter.shared.reloadTimelines(ofKind: "CaffeineWidget")   // targeted; cheaper

// Inspect what's installed (e.g. only fetch data for configured widgets):
WidgetCenter.shared.getCurrentConfigurations { result in
    if case .success(let widgets) = result { /* [WidgetInfo]: kind, family, configuration */ }
}
```

- Prefer `reloadTimelines(ofKind:)` over `reloadAllTimelines()` so you don't refresh unrelated widgets.
- A reliable pattern: when the **app enters the background**, if data changed while it was foreground, call a reload so the widget reflects the latest state. Don't reload on every trivial change - you share the budget.
- These calls *request* a reload; the system still schedules it. They are not synchronous renders.
- `getCurrentConfigurations` lets the app fetch data only for widgets the user actually installed (and which configurations), avoiding wasted work.

For server-driven freshness without the app running, use **push-updated widgets** (`WidgetPushHandler`, iOS 26) - see `relevance-and-smart-stacks.md`. For data-loading patterns (network, images, database) in the provider, see `data-and-networking.md`.

## Entry relevance (for Smart Stack rotation)

A `TimelineEntry` can carry an optional `relevance` that influences when the Smart Stack surfaces this widget:

```swift
struct Entry: TimelineEntry {
    let date: Date
    let value: Value
    var relevance: TimelineEntryRelevance? = nil
}

// score is a Float, RELATIVE to your own entries only; <= 0 means "do not surface"
Entry(date: d, value: v, relevance: TimelineEntryRelevance(score: 0.8, duration: 600))
```

- Higher `score` makes the system more likely to rotate this widget to the top of a Smart Stack at that entry's time. Score is meaningful only relative to the other entries *you* have provided.
- `score <= 0` means "nothing worth showing - don't rotate to me".
- `duration: 0` means the score holds until your next relevance-bearing entry.

For richer, context-driven relevance (location, date ranges, the watchOS Smart Stack), see `relevance-and-smart-stacks.md`.

## Checklist

- `placeholder` synchronous; `snapshot` realistic and populated; read `context.family`/`isPreview`.
- For configurable widgets, `AppIntentTimelineProvider` with the **intent first**, async; add `recommendations()` for watchOS/StandBy.
- `TimelineReloadPolicy` is a struct; `.after(_:)` takes a `Date`.
- Cover a window with **multiple entries**; use self-updating `Text`/`Gauge`/`ProgressView` for live values - don't reload to animate.
- Respect the **40-70/day budget**; reload targeted (`ofKind:`) and on app-background; lean on the budget-free reload triggers.
- Use `WidgetPushHandler` (iOS 26) for server-driven updates, not a tight `.after` loop.
