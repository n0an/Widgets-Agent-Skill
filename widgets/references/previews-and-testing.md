# Previews, testing, and debugging

You can iterate on widgets entirely in previews, across families, color schemes, and rendering modes, without running the host app. Most "it won't update" reports are the **reload budget**, not a bug.

## The `#Preview(as:)` macro (iOS 17+)

The modern preview macro renders a real widget with a timeline. The overloads:

```swift
// StaticConfiguration + a timeline provider:
#Preview(as: .systemSmall) { CaffeineWidget() } timelineProvider: { Provider() }

// StaticConfiguration + explicit entries (best for stepping through transitions):
#Preview("Populated", as: .systemMedium) {
    CaffeineWidget()
} timeline: {
    Entry(date: .now, value: .sample)
    Entry(date: .now.addingTimeInterval(3600), value: .later)
}

// AppIntentConfiguration: pass the configuration intent with `using:`
#Preview(as: .accessoryRectangular, using: ConfigurationIntent()) {
    ColorWidget()
} timelineProvider: { Provider() }

// Live Activity: `.content` (or `.dynamicIsland`) + content states
#Preview("Activity", as: .content, using: DeliveryAttributes.preview) {
    DeliveryLiveActivity()
} contentStates: {
    DeliveryAttributes.ContentState.preparing
    DeliveryAttributes.ContentState.onTheWay
}

// Relevance widget (watchOS): relevance entries
#Preview(as: .accessoryRectangular) { HappyHoursWidget() } relevanceEntries: { /* entries */ }
```

- The **first** argument is an optional name; `as:` is the `WidgetFamily` (or `.content` / `.dynamicIsland` for Live Activities).
- Add multiple `#Preview` blocks to see several families/configs at once.
- Stepping through multiple `timeline:` entries lets you watch the iOS 17 transition animations without waiting for real time (see `animations-and-transitions.md`).

Legacy path (pre-iOS 17 / `PreviewProvider`): wrap the entry view with `.previewContext(WidgetPreviewContext(family: .systemSmall))`. Superseded by the macro - see `migration-and-deprecations.md`.

## What to preview

- **Every supported family** - small/medium/large/accessory each in its own preview.
- **Full color AND accented/tinted/vibrant** - the tinted Lock Screen and accent group are where layouts break; preview them with `.environment(\.widgetRenderingMode, .accented)` (and `.vibrant`) on a factored-out view.
- **Light and dark** - `.environment(\.colorScheme, .dark)`.
- **Empty / placeholder / redacted** - confirm `placeholder` and the gallery `snapshot` look right with no real data, and that the placeholder mirrors the loaded layout (`.redacted(reason: .placeholder)`).
- **Live Activity states** - each `ContentState`, plus the Dynamic Island expanded / compact / minimal presentations.
- **Background removed** - simulate Lock Screen/StandBy by checking the `showsWidgetContainerBackground == false` layout.

## The reload budget (the #1 "won't update" cause)

A widget gets a **limited daily budget** of timeline reloads (roughly 40-70/day, scaled by visibility and conditions). When it's spent, the widget freezes until the budget refills. Before assuming a bug:

- Enable **WidgetKit developer mode**: Settings ▸ Developer ▸ enable the widget reload option (present on developer/debug builds). This **strips the reload-budget policies** so you can verify your reload logic actually fires. Never rely on this frequency in production reasoning.
- Confirm you're not polling with a tight `.after(date)` loop that exhausts the budget by mid-day.
- Remember the **budget-free** triggers (app foreground, significant location change, presentation-environment change) - if your widget can lean on those, it stays fresh without spending budget (see `timeline-provider.md`).
- Confirm `WidgetCenter.shared.reloadTimelines(ofKind:)` uses the **exact `kind`** string (a typo silently reloads nothing).

## Debugging checklist for "shows stale / wrong / no data"

1. **App Group wired on both targets?** Same group id on app + extension; reading the same suite. Reading `UserDefaults.standard` in the extension is the classic cause of blank/stale widgets.
2. **`containerBackground` present?** Without it the widget can be blank in the gallery and fails to render in some contexts.
3. **`getSnapshot` returns populated data?** An empty snapshot makes the gallery preview look broken and users won't add it.
4. **Reload actually requested?** After an interactive intent or an app-side data change, is `WidgetCenter.shared.reloadTimelines(ofKind:)` called *after* the write is committed (`context.save()`)?
5. **Right process for writes?** An interactive intent runs in the **app** process; if you wrote to the extension's sandbox it won't be visible. (27 releases: pin write intents to `.main`.)
6. **Budget exhausted?** Test with developer mode on; if it updates there but not in production, you're over budget - reduce reload frequency or move to push.
7. **Provider completes?** A provider that never calls its completion (or never returns from the async method), e.g. on a network error, hangs the widget. Always complete with last-good/placeholder data.
8. **Crash in timeline generation?** A throw/crash in `getTimeline` leaves the last-rendered snapshot; the extension crashes separately from the app - check its logs in Console / device crash logs.

## Live Activity debugging

- Drive `request`/`update` from the app and watch on a real device (the simulator supports Live Activities, but Dynamic Island fidelity and push behavior are best verified on device).
- For push, the relevant log processes are `liveactivitiesd`, `apsd`, and `chronod`; verify `apns-push-type: liveactivity`, the `apns-topic` suffix, and that `content-state` was encoded with a **plain** `JSONEncoder`.
- Confirm `NSSupportsLiveActivities` (and `…FrequentUpdates` for high-frequency) in the **app** Info.plist, and read the token from `pushTokenUpdates` (async), never synchronously.

## Running the extension

Select the **widget extension scheme** and run it; Xcode attaches to the extension process and lets you pick which widget/family/control to launch. Breakpoints in the provider hit during timeline generation.

## Checklist

- Iterate with `#Preview(as:)`; cover every family, full-color **and** accented/vibrant, light/dark, empty/placeholder, and background-removed.
- "Won't update" is usually the **reload budget** - verify with WidgetKit developer mode before debugging code.
- Verify App Group, `containerBackground`, populated snapshot, exact `kind`, post-write reload, the write-process, and that the provider always completes.
- Live Activities: preview each `ContentState`; verify push (topic, push type, plain JSON) and Dynamic Island on a real device.
