# Widget families, sizing, and layout

A widget declares which families it supports, and its view switches on the current family. Shipping a family you didn't lay out looks broken, so support only what you design.

## `WidgetFamily` cases and where they live

| Case | Surface | Introduced |
|---|---|---|
| `.systemSmall` | Home Screen, Today, StandBy (one-up) | iOS 14 |
| `.systemMedium` | Home Screen, Today | iOS 14 |
| `.systemLarge` | Home Screen, Today | iOS 14 |
| `.systemExtraLarge` | iPad Home Screen / Today (iPad only on iOS) | iOS 15 |
| `.systemExtraLargePortrait` | vertical extra-large | visionOS 26; iOS/iPadOS/macOS in the 27 releases |
| `.accessoryCircular` | Lock Screen, watch complication | iOS 16 / watchOS 9 |
| `.accessoryRectangular` | Lock Screen, watch complication | iOS 16 / watchOS 9 |
| `.accessoryInline` | Lock Screen (above the clock), watch | iOS 16 / watchOS 9 |
| `.accessoryCorner` | watch face corner | watchOS 9 only (no iOS) |

Accessory families are **iOS 16**, not 17 - a frequent off-by-one. `.accessoryCorner` is watch-only; don't list it for an iOS widget.

## The per-family switch

Read `@Environment(\.widgetFamily)` and branch. Keep each branch a deliberately designed layout, not a scaled copy:

```swift
struct MyWidgetView: View {
    @Environment(\.widgetFamily) private var family
    var entry: Entry

    var body: some View {
        switch family {
        case .systemSmall:        SmallView(entry: entry)
        case .systemMedium:       MediumView(entry: entry)
        case .systemLarge:        LargeView(entry: entry)
        case .accessoryCircular:  CircularView(entry: entry)
        case .accessoryRectangular: RectangularView(entry: entry)
        case .accessoryInline:    Label(entry.title, systemImage: "mic.fill")
        default:                  SmallView(entry: entry)   // future-proof
        }
    }
}
```

Always have a `default`/`@unknown default` branch - new families ship over time and an unhandled family renders empty.

`supportedFamilies` must list exactly these:

```swift
.supportedFamilies([.systemSmall, .systemMedium, .systemLarge, .accessoryCircular, .accessoryRectangular, .accessoryInline])
```

## Sizing: don't hardcode dimensions

Widget point sizes differ across devices and grow each year - never hardcode pixel sizes. Build relative layouts:

- Use `GeometryReader` sparingly (it's costly and forces eager layout); prefer stacks, `Spacer`, `.frame(maxWidth:.infinity, maxHeight:.infinity)`, and `.layoutPriority`.
- `ViewThatFits` picks the first child layout that fits - great for "show the rich layout if there's room, else a compact one":

  ```swift
  ViewThatFits {
      DetailedRow(entry: entry)   // tried first
      CompactRow(entry: entry)    // fallback when space is tight
  }
  ```

- Scale type with Dynamic Type in mind; widgets respect the user's text size.

## Content margins

By default the system applies **content margins** so your content isn't clipped near the rounded corners. When you want full-bleed art (a gradient, a photo, a mesh background) and will manage padding yourself, disable them on the configuration and re-add inner padding where text lives:

```swift
StaticConfiguration(kind: kind, provider: Provider()) { entry in
    MyView(entry: entry)
        .containerBackground(for: .widget) { fullBleedGradient }   // bg ignores margins anyway
}
.contentMarginsDisabled()
```

`containerBackground` always extends edge-to-edge regardless of content margins; `contentMarginsDisabled()` is about your *foreground* content.

On modern OSes, **content margins replaced the widget safe area** - `ignoresSafeArea` no longer has any effect inside a widget. To go edge-to-edge you must use `contentMarginsDisabled()`, then re-apply the default margins where you still want them by reading the environment:

```swift
@Environment(\.widgetContentMargins) var margins
// ...
content.padding(margins)        // re-add default insets around the text
```

When the system removes a widget's container background (Lock Screen, StandBy, watch Smart Stack), `widgetContentMargins` automatically shrink so content goes edge-to-edge. If your widget has no meaningful background to remove (a full-bleed photo or map), opt out of removal with `.containerBackgroundRemovable(false)` on the configuration.

## `ContainerRelativeShape`

To round an inner element so its corners stay **concentric** with the widget's (or the Lock Screen container's) corner radius - which you don't know and shouldn't hardcode - use `ContainerRelativeShape`:

```swift
.background {
    ContainerRelativeShape()
        .fill(LinearGradient(colors: [.white.opacity(0.3), .clear], startPoint: .bottom, endPoint: .top))
}
```

This is especially useful in accessory widgets to fill the system's circular/rounded container exactly.

## Accessory family layout notes

- `accessoryInline` is a single line of text + optional `systemImage` (a `Label`); styling is mostly system-controlled.
- `accessoryCircular` is a small circle - one glyph, a gauge, or a tiny progress ring.
- `accessoryRectangular` is roughly three lines of text - a small dashboard.
- All accessory families render in a **tint/vibrant mode**, never full color on the Lock Screen - design them as monochrome-friendly. See `rendering-modes-and-tinting.md` and `lock-screen-and-watch.md`.

## Checklist

- Support only families you actually lay out; switch on `\.widgetFamily` with a `default`.
- Accessory families are **iOS 16+**; `.accessoryCorner` is watch-only.
- Never hardcode sizes; use relative layout, `ViewThatFits`, `ContainerRelativeShape`.
- `contentMarginsDisabled()` for full-bleed foreground; `containerBackground` is always edge-to-edge.
