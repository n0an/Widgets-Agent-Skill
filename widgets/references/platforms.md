# Platforms: visionOS, CarPlay, macOS, watchOS, and the availability map

The same widget code reaches many surfaces, but a few APIs are platform-specific and several features arrived on different platforms in different years. Gate carefully.

## visionOS spatial widgets (visionOS 26 / the 26 releases)

iPhone/iPad widgets become available on visionOS automatically, rendered as **spatial objects** in the user's space. Three modifiers tune the spatial presentation:

```swift
StaticConfiguration(kind: "weather", provider: Provider()) { WeatherView(entry: $0) }
    .supportedMountingStyles([.recessed])        // .elevated (floats) / .recessed (inset into a wall)
    .widgetTexture(.paper)                        // .glass (default) / .paper (poster look) - visionOS only
    .supportedFamilies([.systemExtraLargePortrait])
```

- `supportedMountingStyles([WidgetMountingStyle])` - `.elevated` and/or `.recessed`; default is both. (iOS/visionOS 26.)
- `widgetTexture(WidgetTexture)` - `.glass` or `.paper`; **visionOS only**.
- `@Environment(\.levelOfDetail)` → `LevelOfDetail` (`.default` / `.simplified`) - on visionOS it changes with the user's **proximity** (far away → `.simplified`), animating like a timeline change; on other platforms it's always `.default`. Use it to hide controls / enlarge type when the widget is viewed from across the room:

  ```swift
  @Environment(\.levelOfDetail) var levelOfDetail
  var body: some View {
      TotalView().font(levelOfDetail == .simplified ? .largeTitle : .title)
      if levelOfDetail == .default { LogButton() }   // hide interactive bits when far
  }
  ```

## CarPlay (iOS 26)

- **Widgets in CarPlay**: from iOS 26, all cars (previously CarPlay Ultra only) can show widgets, rendered StandBy-style - `systemSmall`, `.fullColor`, background removed. Adapt as you would for StandBy; interactions work on touchscreens.
- **Live Activities in CarPlay**: surfaced automatically on the CarPlay Dashboard from a paired iPhone; adopt `supplementalActivityFamilies([.small])` and branch on `\.activityFamily` so the small presentation fits.

## macOS

- Widgets run on the desktop and in Notification Center; the same families apply (no accessory families on the Mac).
- **Controls** reach macOS Control Center and the menu bar in the **26 releases** (not 2024).
- **Live Activities** from a paired iPhone (running iOS 18+) appear in the **macOS menu bar** automatically - no Mac code needed.

## watchOS

- The watch widget runtime mirrors iOS (minus core count). An iOS Lock Screen / accessory widget largely "just works" on watchOS 27, possibly needing minor layout tweaks.
- Accessory families map to complication slots; `.accessoryCorner` is **watch-only**.
- Refresh paths: timeline reloads + **APNs-based** refresh (watchOS 26) + **watch-connectivity-based** refresh (the 27 releases). Budget guidance is ~50 widget/complication updates/day; watch-face complications get the top tier.
- Controls and Live Activities from a paired iPhone are auto-forwarded to the watch Smart Stack.
- `RelevanceConfiguration` (show-only-when-relevant widgets) is **watchOS 26+** - see `relevance-and-smart-stacks.md`.

## tvOS - not a WidgetKit platform

There are **no WidgetKit widgets on tvOS.** The `WidgetFamily` / `Widget` APIs are unavailable there (availability is iOS, iPadOS, Mac Catalyst, macOS 11+, visionOS 26+, watchOS 9+ - no tvOS). The tvOS equivalent of "glanceable content from your app" is the **Top Shelf** (the `TVTopShelf` framework / `TVTopShelfContentProvider`), which is a separate API with its own content model - not a widget. If a task asks for a "tvOS widget", it means Top Shelf, which is out of scope for this skill.

## New widget family

- `.systemExtraLargePortrait` - a vertically-oriented extra-large family; **visionOS 26**, broadening to iOS/iPadOS/macOS in the **27 releases**. Add via `supportedFamilies` behind the right availability.

## Availability map (gate on these)

| Feature | iOS | watchOS | macOS | visionOS |
|---|---|---|---|---|
| Widgets, timelines | 14 | 9 | 11 | 1 |
| `systemExtraLarge` | 15 (iPad) | - | 11 | - |
| Accessory families | 16 | 9 | - | - |
| Interactive widgets (`Button/Toggle(intent:)`) | 17 | 10 | 14 | 1 |
| `AppIntentConfiguration` | 17 | 10 | 14 | 1 |
| `widgetAccentedRenderingMode` (Image) | 18 | 10 | 15 | 26 |
| `supplementalActivityFamilies` / `activityFamily` | 18 | - | - | - |
| Controls (`ControlWidget`) | 18 | 26 | 26 | - |
| Push-updated widgets (`WidgetPushHandler`) | 26 | 26 | 26 | 26 |
| `RelevanceConfiguration` | - | 26 | - | - |
| Spatial modifiers (`supportedMountingStyles`, `levelOfDetail`) | 26 | - | - | 26 |
| `widgetTexture` | - | - | - | 26 |
| `systemExtraLargePortrait` | 27 | - | 27 | 26 |
| Landscape Dynamic Island (`isDynamicIslandLimitedInWidth`) | 27 | - | - | - |
| `allowedExecutionTargets` (widget intents) | 27 | 27 | 27 | 27 |

Live Activities (`ActivityKit`) are iOS 16.1+; push-to-start `pushToStartToken` is iOS 17.2+.

## Naming

Call the WWDC 2025 cohort **"the 26 releases"** (iOS 26 / watchOS 26 / …) and the WWDC 2026 cohort **"the 27 releases"** (iOS 27 / …). Do **not** call them "iOS 19".

## Checklist

- visionOS: `supportedMountingStyles`, `widgetTexture` (visionOS-only), `levelOfDetail` for proximity.
- CarPlay/StandBy: design a `systemSmall`, full-color, background-removed variant; `supplementalActivityFamilies([.small])` for Live Activities.
- macOS: no accessory families; Controls and Live-Activity-in-menu-bar are 26.
- watchOS: accessory == complication; `RelevanceConfiguration` is watchOS 26; mind the refresh budget.
- Gate every newer API on the table above; use the "26/27 releases" naming.
