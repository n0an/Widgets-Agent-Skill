# Deep linking and navigation from widgets

A tap on a widget either runs an `AppIntent` (no app launch - see `interactive-widgets.md`) or **opens the app** to a destination. This file is the open-the-app path: `widgetURL`, `Link`, routing on the app side, and the per-surface rules.

## `widgetURL` vs `Link`

- **`widgetURL(_:)`** - one URL for the whole widget (per family/view). Required for `systemSmall` and all accessory families, which have exactly **one tap target**.
- **`Link(destination:)`** - multiple tappable regions inside `systemMedium` / `systemLarge` / `systemExtraLarge`, each with its own URL. Ignored on small/accessory families.

```swift
// Small / accessory: whole-widget target.
SmallView(entry: entry)
    .widgetURL(URL(string: "caffeine://today"))

// Medium / large: per-region links.
HStack {
    Link(destination: URL(string: "caffeine://streak")!) { StreakView() }
    Link(destination: URL(string: "caffeine://log")!)    { LogView() }
}
```

Precedence: if a `Link` is tapped, its URL wins; taps outside any `Link` fall back to the widget's `widgetURL`. A small widget should set `widgetURL` to the content it displays, so the tap lands exactly where the user expects.

## Designing the URL scheme

Use a custom scheme (or a universal/https link you already handle) and encode the destination precisely so the app navigates straight to the shown content:

```swift
// Encode the entity id so the app opens that exact item.
.widgetURL(URL(string: "caffeine://drink/\(entry.drink.id)"))
```

- Prefer **stable identifiers** over indices.
- If your app already handles **universal links**, reuse them - the same routing works from the widget and from the web.
- Keep one consistent scheme across all widgets so the app's router stays simple.

## Handling the URL in the app

### SwiftUI

```swift
@main
struct MyApp: App {
    @State private var router = Router()
    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(router)
                .onOpenURL { url in router.handle(url) }   // widget taps arrive here
        }
    }
}
```

`onOpenURL` fires whether the app was suspended or cold-launched. Route on the URL's host/path:

```swift
func handle(_ url: URL) {
    switch url.host() {
    case "streak":  path = [.streak]
    case "drink":   path = [.drink(id: url.lastPathComponent)]
    default:        break
    }
}
```

### UIKit

Implement `scene(_:openURLContexts:)` (or `application(_:open:options:)` without scenes) and route from there. With a scene already running, the URL arrives at the scene delegate; on cold launch it's in the connection options.

## Live Activities

A Live Activity's Lock Screen view and Dynamic Island can deep-link too:

```swift
// Whole Live Activity / Dynamic Island target:
.widgetURL(context.attributes.deepLinkURL)
// Tint the Dynamic Island keyline to match your brand:
.keylineTint(.orange)
```

Tapping the Dynamic Island or banner opens the app via this URL; combine with `Button(intent:)`/`LiveActivityIntent` for in-place actions (see `live-activities.md`).

## Controls

A Control that should **launch the app** uses an `OpenIntent` rather than a URL:

```swift
struct OpenTimerControl: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: "OpenTimer") {
            ControlWidgetButton(action: OpenTimerIntent()) {
                Label("Open Timer", systemImage: "timer")
            }
        }
    }
}

struct OpenTimerIntent: OpenIntent {
    static let title: LocalizedStringResource = "Open Timer"
    @Parameter(title: "Screen") var target: AppScreen
}
```

The `OpenIntent`'s type must be a member of **both** the app and the widget-extension targets. See `controls.md`.

## Common mistakes

- **A `Link` in a `systemSmall` widget** - ignored. Small widgets get exactly one target; use `widgetURL`.
- **No `widgetURL` on a small/accessory widget** - the tap does nothing. Always provide one.
- **Opening the app for an action that shouldn't need it** (toggle, increment, mark done) - use an `AppIntent` button instead, which keeps the user on the Home Screen.
- **Generic deep link** (just opens the app's root) - encode the specific destination so the tap lands on the shown content.

## Checklist

- `widgetURL` for small/accessory (one target); `Link` for medium/large (many targets).
- Encode the exact destination (stable id); reuse universal links if you have them.
- Handle on the app side with `onOpenURL` (SwiftUI) / scene delegate (UIKit); works on cold launch and resume.
- Live Activities: `widgetURL` + `keylineTint`; Controls: `OpenIntent` (in both targets).
- For state changes, prefer an `AppIntent` over opening the app.
