# Relevance, Smart Stacks, and push-updated widgets

Two related but distinct problems: getting your widget to **surface at the right moment** in a Smart Stack, and keeping it **fresh from the server** without burning the reload budget.

## Smart Stack rotation: entry relevance

The Smart Stack rotates widgets to the top based on relevance signals. The lightest lever is per-entry relevance on a normal timeline (iOS 14+):

```swift
struct Entry: TimelineEntry {
    let date: Date
    let value: Value
    var relevance: TimelineEntryRelevance?
}

Entry(date: gameStart, value: v,
      relevance: TimelineEntryRelevance(score: 0.9, duration: 7200))   // score is Float
```

- Higher `score` = more likely to be rotated to the top at that entry's time.
- `duration` (seconds) is how long the entry stays relevant.

## `relevance()` with `WidgetRelevance` (iOS 18+)

For richer, context-driven hints, implement `relevance()` on your provider, returning a `WidgetRelevance` made of `WidgetRelevanceAttribute`s, each tied to a `RelevantContext` (date range, location, etc.). This is what powers contextual Smart Stack suggestions, especially on watchOS.

```swift
func relevance() async -> WidgetRelevance<Void> {                 // or WidgetRelevance<Intent> for AppIntent providers
    let attributes = upcomingEvents.map { event in
        WidgetRelevanceAttribute(
            context: .date(range: event.start...event.end, kind: .activity)   // .date(range:kind:) is iOS 26+
        )
    }
    return WidgetRelevance(attributes)
}
```

`RelevantContext` factories (RelevanceKit; functional mainly on watchOS) include `.date(_:)`, `.date(interval:kind:)`, `.date(range:kind:)` (iOS 26+), `.location(_:)`, `.location(category:)`, `.sleep(_:)`, `.fitness(_:)`, `.hardware(headphones:)`. Use these to say "this matters at this place / time / activity".

## Relevance widgets: `RelevanceConfiguration` (watchOS 26+)

WWDC 2025 added a whole configuration type whose widget appears in the watch **Smart Stack only when it's relevant**, and can show multiple instances for multiple relevant configurations (e.g. a "happy hour" widget for each nearby shop currently in happy hour). It uses a `RelevanceEntriesProvider` instead of a `TimelineProvider`:

```swift
struct HappyHoursWidget: Widget {
    var body: some WidgetConfiguration {
        RelevanceConfiguration(kind: "HappyHours", provider: HappyHoursProvider()) { entry in
            HappyHourView(entry: entry)
        }
    }
}

struct HappyHoursProvider: RelevanceEntriesProvider {
    func placeholder(context: Context) -> HappyHourEntry { .placeholder }

    func relevance() async -> WidgetRelevance<ShopHappyHour> {
        let attributes = fetchHappyHours().map {
            WidgetRelevanceAttribute(
                configuration: $0,
                context: .date(range: $0.start...$0.end, kind: .activity)
            )
        }
        return WidgetRelevance(attributes)
    }

    func entry(configuration: ShopHappyHour, context: Context) async throws -> HappyHourEntry {
        HappyHourEntry(shop: configuration.shop, range: configuration.range)
    }
}
```

Key contrasts with a timeline provider:
- It returns a **single entry per configuration** via `entry(configuration:context:)` (`async throws`), not a timeline.
- `relevance()` decides **which configurations** are relevant **and when**; the system instantiates the widget for each.
- `RelevanceConfiguration` / `RelevanceEntriesProvider` are **watchOS 26+**.

Smart Stack etiquette (watchOS): don't camp the top slot (users dismiss apps that do); alert selectively; remove the activity/relevance once the real-world event ends. Watch-face complications sit in the top refresh-budget tier; Smart Stack widgets refresh in proportion to how often the user actually looks.

## Preconfigured widgets: `recommendations()`

Distinct from relevance: on surfaces with no configuration editor (watchOS Smart Stack and complications, StandBy), the system needs preconfigured instances of your configurable widget. Provide them from the timeline provider's `recommendations()`:

```swift
func recommendations() -> [AppIntentRecommendation<ConfigurationIntent>] {
    City.popular.map { AppIntentRecommendation(intent: ConfigurationIntent(city: $0), description: $0.name) }
}
```

- On **watchOS this is effectively required** for a configurable widget; return `[]` where an editor exists.
- Call `WidgetCenter.shared.invalidateConfigurationRecommendations()` when the set changes.
- The legacy `IntentTimelineProvider` returns `[IntentRecommendation]`; the modern `AppIntentTimelineProvider` returns `[AppIntentRecommendation<Intent>]`.

> Legacy note: pre-iOS-18 Smart Stack rotation was driven by SiriKit donations (`INRelevantShortcut`, `INInteraction.donate()`). That donation path is superseded by App Intents relevance; `TimelineEntryRelevance` survives. See `migration-and-deprecations.md`.

## Push-updated widgets: `WidgetPushHandler` (iOS 26+)

To refresh a widget from your server *without* the app running and without polling, register a push handler on the configuration. APNs tells WidgetKit to reload that widget's timelines - like a remote `reloadAllTimelines()`.

```swift
import WidgetKit

struct CaffeinePushHandler: WidgetPushHandler {
    func pushTokenDidChange(_ pushInfo: WidgetPushInfo, widgets: [WidgetInfo]) {
        // Send pushInfo.token (+ which widgets) to your server for targeting.
        Server.register(token: pushInfo.token, widgets: widgets)
    }
}

struct CaffeineWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "CaffeineWidget", provider: Provider()) { entry in
            CaffeineWidgetView(entry: entry)
        }
        .pushHandler(CaffeinePushHandler.self)        // pass the TYPE (.self), not an instance
    }
}
```

Setup:
- Add the **Push Notifications** capability to the **widget extension** target (you get the token through WidgetKit, not UserNotifications).
- `pushHandler(_:)` takes the handler **type**.
- APNs request to trigger a reload:

  ```
  :method = POST
  :path = /3/device/<DEVICE_TOKEN>
  apns-push-type = widgets
  apns-topic = <bundleID>.push-type.widgets
  { "aps": { "content-changed": true } }
  ```

- It's **budgeted and opportunistic**, not instant, and **broadcast/channel push is not supported** for widgets (that's a Live Activity feature). Use WidgetKit developer mode to bypass budgets while testing.

## Choosing the freshness mechanism

```
Urgent, needs the user's attention now?        -> User Notification
Ongoing event the user watches for a while?    -> Live Activity (token/channel push)
Keep a glanceable widget fresh from server?    -> Push-updated widget (WidgetPushHandler)
Predictable schedule, no server?               -> Timeline reload policy (.after / .atEnd)
```

## Checklist

- Per-entry rotation hint: `TimelineEntryRelevance(score:duration:)` (`score` is `Float`).
- Contextual relevance: `relevance()` → `WidgetRelevance([WidgetRelevanceAttribute(context:)])`; `RelevantContext.date(range:kind:)` is iOS 26+.
- watchOS "show only when relevant" widget: `RelevanceConfiguration` + `RelevanceEntriesProvider` (watchOS 26+), one entry per configuration.
- Server-driven refresh: `WidgetPushHandler` + `.pushHandler(Type.self)` (iOS 26+), Push capability on the **extension**, `apns-push-type: widgets`; budgeted, no broadcast.
