# Worked example: a Quick Actions widget, end to end

This builds one real widget from scratch, touching every concern in the right order: the entry and provider, the per-family layout, multiple deep links, full-color-vs-tinted handling, an animation that works without a live view, content margins, and the preview. The code is adapted from the **VivaDicta** iOS app's Quick Actions widget - a `systemMedium` widget with three tappable pills (Search, Record, Ask AI), where the Record pill has an animated mesh-gradient background.

It's deliberately a `StaticConfiguration` widget that opens the app via deep links (no configuration, no interactive intents) - the most common shape. Cross-references point to the deeper reference for each step.

## 1. The entry

The entry carries the date plus one extra value: a `Float` time parameter `t` that drives the background animation. A widget can't run `TimelineView`, so we derive the animation phase from the timeline instead (see step 5).

```swift
import WidgetKit
import SwiftUI

struct QuickActionsWidgetEntry: TimelineEntry {
    let date: Date
    /// Mesh-gradient time parameter: monotonic hour-offset from timeline boot,
    /// so the mesh drifts without snapping at midnight.
    let t: Float
}
```

## 2. The provider

`placeholder` is synchronous; `getSnapshot` returns realistic data for the gallery; `getTimeline` emits **one entry per hour for 24 hours** and reloads at the end. Emitting many entries lets the background drift over the day without spending reloads (see `timeline-provider.md`).

```swift
struct QuickActionsWidgetProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuickActionsWidgetEntry {
        QuickActionsWidgetEntry(date: .now, t: 0)
    }

    func getSnapshot(in context: Context, completion: @escaping (QuickActionsWidgetEntry) -> Void) {
        completion(QuickActionsWidgetEntry(date: .now, t: 0))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<QuickActionsWidgetEntry>) -> Void) {
        let now = Date()
        let entries = (0..<24).compactMap { hourOffset in
            Calendar.current.date(byAdding: .hour, value: hourOffset, to: now)
                .map { QuickActionsWidgetEntry(date: $0, t: Float(hourOffset)) }
        }
        let reloadDate = now.addingTimeInterval(24 * 3600)
        completion(Timeline(entries: entries, policy: .after(reloadDate)))
    }
}
```

## 3. The widget declaration

`StaticConfiguration`, one supported family, gallery copy, and `contentMarginsDisabled()` because the pills manage their own padding and we want them edge-to-edge.

```swift
struct VivaDictaQuickActionsWidget: Widget {
    let kind = "VivaDictaQuickActionsWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuickActionsWidgetProvider()) { entry in
            QuickActionsWidgetEntryView(entry: entry)
        }
        .configurationDisplayName("Quick Actions")
        .description("Search notes, start a recording, or open Ask AI.")
        .supportedFamilies([.systemMedium])
        .contentMarginsDisabled()
    }
}
```

## 4. The layout: multiple deep links

A `systemMedium` widget can have several tap targets, each a `Link` with its own URL. Here: a full-width Search pill on top, and a Record + Ask row below. The whole thing sets its background with `containerBackground(for: .widget)`. (See `deep-linking-and-navigation.md`.)

```swift
struct QuickActionsWidgetEntryView: View {
    @Environment(\.widgetRenderingMode) private var renderingMode
    @Environment(\.colorScheme) private var colorScheme
    var entry: QuickActionsWidgetEntry

    private var isFullColor: Bool { renderingMode == .fullColor }

    var body: some View {
        VStack(spacing: 8) {
            Link(destination: URL(string: "openSearchFromWidget")!) { searchPill }
                .frame(maxWidth: .infinity)

            HStack(spacing: 8) {
                Link(destination: URL(string: "startRecordFromWidget")!) { recordPill }
                    .frame(maxWidth: .infinity)
                Link(destination: URL(string: "openAskFromWidget")!) { askPill }
                    .frame(maxWidth: .infinity)
            }
        }
        .padding(12)
        .containerBackground(for: .widget) { backgroundGradient }
    }

    private var backgroundGradient: LinearGradient {
        colorScheme == .dark
            ? LinearGradient(colors: [Color(red: 0.08, green: 0.06, blue: 0.12),
                                      Color(red: 0.18, green: 0.12, blue: 0.28)],
                             startPoint: .topLeading, endPoint: .bottomTrailing)
            : LinearGradient(colors: [.white, Color(red: 0.88, green: 0.85, blue: 0.95)],
                             startPoint: .topLeading, endPoint: .bottomTrailing)
    }
}
```

## 5. Full color vs tinted, and animating without a live view

Two things happen per pill:

- **Color-mode awareness.** In `.fullColor` the pills get real backgrounds; in `.accented`/`.vibrant` the backgrounds collapse to `.clear` so the system tint takes over, and the text/icons are pushed into the accent group with `.widgetAccentable()`. (See `rendering-modes-and-tinting.md`.)
- **Animation from the entry.** The Record pill's mesh-gradient background takes the entry's `t`. Because the provider emitted an entry per hour with an increasing `t`, the mesh **drifts across the day** as the system swaps entries - no `TimelineView`, no extra reloads. (See `animations-and-transitions.md`.)

```swift
extension QuickActionsWidgetEntryView {
    private var recordForeground: Color {
        !isFullColor ? .white : (colorScheme == .dark ? .white : .black)
    }
    private var recordFallbackBackground: Color {
        !isFullColor ? .clear : (colorScheme == .dark ? Color(red: 0.32, green: 0.06, blue: 0.08) : .black)
    }

    var recordPill: some View {
        HStack(spacing: 8) {
            Image(systemName: "record.circle.fill").font(.title3).foregroundStyle(.red).widgetAccentable()
            Text("Record").font(.headline).foregroundStyle(recordForeground).widgetAccentable()
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background {
            if isFullColor {
                WidgetRecordPillBackground(cornerRadius: 22, colorScheme: colorScheme, t: entry.t)
            } else {
                recordFallbackBackground   // clear in tinted mode → system tint shows through
            }
        }
        .clipShape(.rect(cornerRadius: 22))
    }
}
```

The animated background derives a `MeshGradient`'s control points from `t` via a sine function - a static gradient per entry, but a moving one across entries:

```swift
struct WidgetRecordPillBackground: View {
    let cornerRadius: CGFloat
    let colorScheme: ColorScheme
    let t: Float                 // supplied by the timeline entry, not a live clock

    var body: some View {
        StaticWidgetMeshGradient(colors: meshColors, t: t)
            .mask(RoundedRectangle(cornerRadius: cornerRadius).stroke(lineWidth: 26).blur(radius: 6))
            .blendMode(colorScheme == .dark ? .lighten : .normal)
            .background(colorScheme == .dark ? Color.black : Color.white)
            .clipShape(RoundedRectangle(cornerRadius: cornerRadius))
    }
    private var meshColors: [Color] { colorScheme == .dark
        ? [.red, .purple, .indigo, .orange, .white, .blue, .yellow, .black, .mint]
        : [.blue, .red, .orange, .orange, .indigo, .red, .cyan, .purple, .mint] }
}

private struct StaticWidgetMeshGradient: View {
    let colors: [Color]; let t: Float
    var body: some View {
        MeshGradient(width: 3, height: 3, points: [
            .init(0, 0), .init(0.5, 0), .init(1, 0),
            [sinInRange(-0.8...(-0.2), offset: 0.439, timeScale: 0.342, t: t),
             sinInRange(0.3...0.7,     offset: 3.42,  timeScale: 0.984, t: t)],
            // ...the remaining mesh points, each a sinInRange(...) of t...
            [sinInRange(1.0...1.5,     offset: 0.939, timeScale: 0.056, t: t),
             sinInRange(1.3...1.7,     offset: 0.47,  timeScale: 0.342, t: t)]
        ], colors: colors)
    }
    private func sinInRange(_ r: ClosedRange<Float>, offset: Float, timeScale: Float, t: Float) -> Float {
        let amp = (r.upperBound - r.lowerBound) / 2, mid = (r.upperBound + r.lowerBound) / 2
        return mid + amp * sin(timeScale * t + offset)
    }
}
```

> `MeshGradient` is iOS 18+. The takeaway pattern is general: **to animate a widget, put a phase value in the entry and emit a series of entries** - the system tweens between the pre-rendered snapshots for free.

## 6. The preview

Step through several `t` values to watch the mesh drift without waiting hours:

```swift
#Preview(as: .systemMedium) {
    VivaDictaQuickActionsWidget()
} timeline: {
    QuickActionsWidgetEntry(date: .now, t: 0)
    QuickActionsWidgetEntry(date: .now, t: 6)
    QuickActionsWidgetEntry(date: .now, t: 12)
    QuickActionsWidgetEntry(date: .now, t: 18)
}
```

## What this example demonstrates

- **Provider discipline** - synchronous placeholder, realistic snapshot, a window of entries with a sensible reload policy (`timeline-provider.md`).
- **Many entries instead of frequent reloads** - the per-hour `t` animates the background within budget.
- **Multiple deep links** in a medium widget, each `Link` opening a different destination (`deep-linking-and-navigation.md`).
- **Color-mode awareness** - real backgrounds in full color, `.clear` + `widgetAccentable()` in tinted/vibrant (`rendering-modes-and-tinting.md`).
- **`contentMarginsDisabled()`** for an edge-to-edge custom layout (`families-and-layout.md`).
- **The preview macro** for fast iteration across entries (`previews-and-testing.md`).

## Variations to reach for next

- Make it **configurable** (let the user pick which three actions): switch to `AppIntentConfiguration` + a `WidgetConfigurationIntent` (`configuration-and-intents.md`).
- Make a pill **act in place** instead of opening the app (e.g. start/stop recording): replace the `Link` with `Button(intent:)` and an `AppIntent` that writes the App Group store and reloads (`interactive-widgets.md`).
- Add a **Lock Screen** variant: support `.accessoryRectangular`/`.accessoryCircular` and design for tint (`lock-screen-and-watch.md`).
- Add a **Control** for one-tap recording from Control Center (`controls.md`).
