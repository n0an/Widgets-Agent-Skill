# Fetching data: networking, persistence, images, and location

A widget's timeline provider has to get data from somewhere - a shared store the app wrote, a network request, a database, the user's location. It does this under tight time and memory limits in a separate process. This file covers the data-loading patterns that keep a widget correct and within budget.

## The golden rule: prefer data the app already wrote

The cheapest, most reliable widget reads pre-computed data from the **App Group** container that the app keeps up to date, and only reaches the network when it must. The app does the heavy lifting (login, large fetches, decoding) and writes a small, widget-ready snapshot; the widget reads it.

```swift
// App side: after refreshing, write a compact widget payload + reload.
let payload = WidgetPayload(caffeineMg: total, lastDrink: name, updated: .now)
let defaults = UserDefaults(suiteName: "group.com.you.app")!
defaults.set(try? JSONEncoder().encode(payload), forKey: "widget.today")
WidgetCenter.shared.reloadTimelines(ofKind: "CaffeineWidget")

// Widget side: read it synchronously in the provider.
func makeEntry() -> Entry {
    let defaults = UserDefaults(suiteName: "group.com.you.app")!
    let payload = (defaults.data(forKey: "widget.today")).flatMap { try? JSONDecoder().decode(WidgetPayload.self, from: $0) }
    return Entry(date: .now, payload: payload ?? .placeholder)
}
```

Use a file in the group container (or a shared `ModelContainer`) for larger data; use the `UserDefaults` suite for small scalars and codable blobs.

## Networking from the timeline provider

When the widget genuinely needs to fetch (it has no app to do it for it, or the data is widget-specific), use `async/await` in the async provider, or `URLSession` with the completion-based provider:

```swift
struct Provider: TimelineProvider {
    func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> Void) {
        Task {
            do {
                let repo = try await NetworkClient.shared.fetchRepo("owner/name")
                let entry = Entry(date: .now, repo: repo)
                let next = Calendar.current.date(byAdding: .hour, value: 6, to: .now)!
                completion(Timeline(entries: [entry], policy: .after(next)))
            } catch {
                // Always complete - never leave the system waiting. Reuse last-good or a placeholder.
                completion(Timeline(entries: [.placeholder], policy: .after(.now.addingTimeInterval(900))))
            }
        }
    }
}
```

Rules:
- **Always call the completion** (or return from the async method), even on error - a provider that never completes hangs the widget and wastes the budget. On failure, fall back to cached/last-good data and a short retry.
- **Keep payloads tiny.** Apple's guidance for background refresh is roughly **under 100 KB per fetch**. Fetch only what the widget shows (a count, a title, a small thumbnail), not full responses.
- **Don't fetch to animate.** If the data only changes on a schedule, fetch once and emit multiple entries; for live counters use `Text(_:style:)`. (See `timeline-provider.md`.)
- **Respect the reload budget.** Network fetches happen on reloads, which are budgeted; a tight `.after` loop that fetches every minute dies by mid-day. For server-pushed freshness use a **push-updated widget** (`WidgetPushHandler`, iOS 26) - see `relevance-and-smart-stacks.md`.

## Background URLSession in the widget extension

The widget extension can own a **background** `URLSession` for transfers that should survive the extension being suspended. Unlike an app, there's no app delegate - the widget configuration handles its own session events:

```swift
StaticConfiguration(kind: kind, provider: Provider()) { entry in
    EntryView(entry: entry)
}
.onBackgroundURLSessionEvents { sessionIdentifier, completion in
    // re-create the matching background session, let its delegate finish, then:
    completion()
}
```

Use this for a larger download that you kick off during timeline generation and finish later; for the common small-JSON case, a plain `async` fetch in the provider is simpler.

## Images: load them into the entry, don't fetch in the view

A widget view is archived and rendered later, so an `AsyncImage` that fetches at render time won't work reliably. Download/resize the image during timeline generation and put the bytes (or a `UIImage`/`Image`) into the entry:

```swift
struct Entry: TimelineEntry {
    let date: Date
    let avatar: Image      // resolved during getTimeline, not fetched in the view
}

// In the provider:
let data = try await URLSession.shared.data(from: avatarURL).0
let image = downsample(data, to: CGSize(width: 120, height: 120))   // shrink BEFORE storing
let entry = Entry(date: .now, avatar: Image(uiImage: image))
```

- **Downsample before storing.** Widget memory limits are small; a full-resolution image can get the extension killed. Resize to the display size.
- Cache decoded images in the App Group container so repeated reloads don't re-download.
- For SF Symbols and bundled assets, just reference them in the view - no fetching needed.

## SwiftData / Core Data in widgets

The widget can read the app's database if the store lives in the App Group container. Use a one-shot fetch on a fresh context in the provider (and in any interactive intent):

```swift
func fetchDays() -> [Day] {
    let context = ModelContext(Persistence.container)     // container URL in the App Group
    let start = Date().startOfMonth, end = Date().endOfMonth
    let descriptor = FetchDescriptor<Day>(
        predicate: #Predicate { $0.date >= start && $0.date < end },
        sortBy: [.init(\.date)])
    return (try? context.fetch(descriptor)) ?? []
}
```

- `@Query` only works inside SwiftUI `View`s in the app - **never** in a provider or intent. Use `FetchDescriptor` + `ModelContext`.
- Configure the `ModelContainer` / Core Data `NSPersistentContainer` with a store URL in `FileManager.default.containerURL(forSecurityApplicationGroupIdentifier:)` so app and widget share one database.
- After an interactive intent writes (`try context.save()`), reload (`WidgetCenter.shared.reloadTimelines(ofKind:)`). In the 27 releases, writes must run in the app process (`allowedExecutionTargets = .main`) - the widget extension's store view is read-only.

## Location in a widget

A widget can use location for context (weather, transit). Opt in and check the widget-specific authorization:

```swift
// Info.plist: NSWidgetUsesLocation = YES  (and a usage-description string)
let manager = CLLocationManager()
if manager.isAuthorizedForWidgetUpdates {        // widget-specific check
    manager.requestLocation()                    // call from the provider; prefer coarse for speed
}
```

- The **"While Using the App or Widgets"** authorization grants the widget location for up to **15 minutes after it was last viewed**.
- A significant-location-change is one of the **budget-free** reload triggers, so location-driven widgets can refresh without spending the timeline budget.
- Prefer coarse/`reducedAccuracy` location for speed and privacy unless you truly need precise.

## Privacy and the Lock Screen

Data shown on the Lock Screen may be visible while the device is locked. Mark sensitive views so the system redacts them when locked:

```swift
Text(account.balance).privacySensitive()         // redacted to the placeholder when locked
```

For full protection, adopt the default-data-protection entitlement (`NSFileProtectionComplete`); the system then swaps active content for the placeholder and withholds updates while the device is passcode-locked.

## Checklist

- Prefer reading an App-Group snapshot the app wrote; fetch from the widget only when necessary.
- In the provider, fetch with `async/await`; **always complete**, fall back to last-good on error, keep payloads < ~100 KB.
- Resolve images during timeline generation and **downsample** before storing; don't `AsyncImage` in the view.
- SwiftData/Core Data: shared store in the App Group, `FetchDescriptor` (never `@Query`) in providers/intents, reload after writes.
- Location: `NSWidgetUsesLocation`, `isAuthorizedForWidgetUpdates`, coarse accuracy; location changes are budget-free reloads.
- `privacySensitive()` for Lock Screen data; server-driven freshness → push-updated widgets, not a fetch loop.
