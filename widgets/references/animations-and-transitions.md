# Animations and transitions in widgets

Since iOS 17, widgets animate. But they animate differently from a normal SwiftUI view: a widget has **no state** and isn't running while it's on screen, so the system animates the **difference between consecutive timeline entries** (and the difference before/after an interaction). Understanding that model is the key to making widgets feel alive without breaking.

## The model: SwiftUI diffs entries

When the visible entry changes - because time advanced to the next entry, or an interactive intent triggered a reload - SwiftUI compares the old and new view trees and animates what changed. You don't drive animations imperatively; you describe two states and let the system tween between them.

- **Recompiling against the iOS 17+ SDK gives you a default implicit spring** and implicit content transitions for free.
- All standard SwiftUI animation, transition, and content-transition APIs are available; they apply across the entry boundary.
- There is **no** `withAnimation { }` on a live tap that you control - the animation happens when the new archived entry is rendered.

## Animating changing values: `contentTransition`

For a number that changes between entries (a count, a score, a total), use a content transition so it rolls instead of snapping:

```swift
Text(total, format: .number)
    .contentTransition(.numericText(value: Double(total)))   // odometer-style roll
```

`.numericText()` is the canonical "counter" transition. `.opacity` and `.interpolate` work for other value changes; `.symbolEffect`-style continuous animations do **not** run in widgets (see below).

## Animating appearing/disappearing content: `transition` + identity

To move a new item in from an edge when it replaces a previous one, bind the view's **identity** to the data so SwiftUI knows it's a new element, then attach a transition and an animation keyed to the value:

```swift
LastDrinkView(drink: entry.latest)
    .id(entry.latest)                                   // identity = "this is a different drink"
    .transition(.push(from: .bottom))
    .animation(.smooth(duration: 0.3), value: entry.latest)
```

Without the `.id(...)`, SwiftUI treats it as the same view with changed content (no transition); with it, the old view leaves and the new one enters via the transition.

## Interactive widgets: the reload is automatic, the toggle is optimistic

When a `Button(intent:)` / `Toggle(intent:)` fires:

- The intent's `perform()` runs, then the system **immediately and reliably reloads** the widget's timeline - this reload is guaranteed (normal scheduled reloads are best-effort). The change you made animates in when the new entry renders.
- A **`Toggle`** flips its presentation **optimistically**, without waiting for the round-trip, because the system pre-renders **both** `isOn` states at archive time. For this to work, a **custom `ToggleStyle` must read `configuration.isOn`** to choose its appearance - if it ignores `configuration.isOn`, the optimistic flip can't render and the toggle feels broken.

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

## Bridging the latency: `invalidatableContent`

There's a brief gap between a tap and the reloaded entry (most visible for an iPhone widget running on a Mac). Mark the views whose value is about to change as invalidatable so the system shows a pending/shimmer treatment until the fresh entry lands:

```swift
Text(total, format: .number).invalidatableContent()
```

The system applies `RedactionReasons.invalidated` during this window; you can read `@Environment(\.redactionReasons)` to style the pending state yourself. Use it **sparingly**, only on the values that actually change from the interaction.

## Live Activities

Live Activities animate the same way - the system diffs `ContentState` updates:

- `.contentTransition(.numericText())` for counting numbers (a score, a countdown reaching a value).
- `.contentTransition(.opacity)` or content-replacement transitions for swapping graphics/state icons.
- Vary the Lock Screen presentation's **height** between moments to draw attention to important changes.
- Use **alerts** (`AlertConfiguration` on `update`) only for genuinely attention-worthy changes - the alert is what lights up the Dynamic Island and (on watch) plays a sound.
- The Dynamic Island's transitions between compact/expanded/minimal are system-driven; keep your regions concentric with the pill so they animate cleanly.

## Animating from the timeline (no `TimelineView`)

`TimelineView(.animation)` does not run in a widget, but you can still produce smooth, continuous motion (a drifting gradient, a slow pulse) by **putting a phase value in the entry and emitting a series of entries**. The system swaps the pre-rendered snapshots over time, so the motion is free - it spends no extra reloads.

The pattern (from VivaDicta's animated mesh-gradient pill background):

```swift
struct Entry: TimelineEntry {
    let date: Date
    let t: Float            // animation phase, increases per entry
}

// Provider: emit one entry per hour, each with a higher t.
let entries = (0..<24).compactMap { hour in
    Calendar.current.date(byAdding: .hour, value: hour, to: now)
        .map { Entry(date: $0, t: Float(hour)) }
}

// View: derive the visual from t (here, MeshGradient control points via sine).
MeshGradient(width: 3, height: 3, points: pointsFrom(t: entry.t), colors: colors)
```

Each rendered entry is a *static* gradient; across entries it drifts. Keep `t` monotonic (an hour-offset from timeline start) rather than wrapping at midnight, so the motion doesn't snap. This is the supported substitute for a continuously animating background. (`MeshGradient` itself is iOS 18+; the technique is general.) See `worked-example.md` for the full widget.

## What does NOT animate

- **`symbolEffect(...)`** - SF Symbol effects (bounce, pulse, variable color animation) are **not supported** in widgets or Live Activities; they silently no-op. Use static symbols and entry-to-entry transitions.
- **Continuous/looping animations**, `TimelineView(.animation)`, video, animated images - none run in a widget. The view is a static snapshot between entries.
- **Gesture-driven animation** - there are no gestures (see `interactive-widgets.md`).
- Self-updating **`Text(_:style:)`** timers/dates DO update live (the system re-renders them) - that's the supported way to show motion without entries or reloads.

## Previewing transitions

Use the `#Preview(as:)` macro with **multiple timeline entries**; Xcode lets you step through them and watch the transitions without waiting for real time to pass:

```swift
#Preview(as: .systemSmall) { MyWidget() } timeline: {
    Entry(date: .now, total: 1)
    Entry(date: .now, total: 2)     // step between these to see the numericText roll
    Entry(date: .now, total: 3)
}
```

## Checklist

- Animations come from **diffing consecutive entries**, not imperative `withAnimation`; recompiling on iOS 17+ gives an implicit spring.
- `contentTransition(.numericText(value:))` for changing numbers; `.transition(...)` + `.id(value)` for replaced content.
- Interactive reload after `perform()` is automatic and guaranteed; custom `ToggleStyle` must read `configuration.isOn`.
- `invalidatableContent()` (sparingly) bridges tap-to-reload latency via `RedactionReasons.invalidated`.
- `symbolEffect`, looping animations, `TimelineView(.animation)`, and video do **not** run; `Text(_:style:)` timers do.
- Preview transitions by stepping through multiple `#Preview` timeline entries.
