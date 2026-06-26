# Lock Screen, StandBy, and watch complications

Accessory widgets (iOS 16+ / watchOS 9+) share one code path: the accessory `WidgetFamily` cases, rendered in a tint/vibrant mode against the wallpaper or watch face. The same widget extension can serve the Lock Screen and, on a paired build, the watch. WidgetKit *added* the accessory families to the existing `WidgetFamily`, so you write them like any other widget and share infrastructure with Home Screen widgets.

## The accessory families

- `.accessoryCircular` - a small circle (a glyph, a gauge, a ring). Lock Screen and watch.
- `.accessoryRectangular` - ~three lines; a tiny dashboard. Lock Screen and watch.
- `.accessoryInline` - one line above the clock / on the watch face; `Label`-style, drawn with **system-defined color and font** (you don't control styling much). On the watch it behaves *as* a widget label.
- `.accessoryCorner` - watch face corner; **watchOS only**. A small circle plus a curved label/gauge.

Branch on `@Environment(\.widgetFamily)` like any widget, and list the accessory families in `supportedFamilies`. A single widget can support both system (Home Screen) and accessory (Lock Screen) families.

## The transparent-container trick (let the wallpaper/face show through)

Accessory widgets should usually let the background show through. The rule that surprises people:

> If you **omit** `containerBackground`, the system adds an **opaque default material**. To get a transparent background, provide an **empty** `containerBackground`.

```swift
struct LockScreenCircularView: View {
    var body: some View {
        Image(systemName: "mic.circle")
            .symbolRenderingMode(.hierarchical)
            .font(.system(size: 40))
            .frame(maxWidth: .infinity, maxHeight: .infinity)
            .background {
                ContainerRelativeShape()
                    .fill(LinearGradient(colors: [.white.opacity(0.2), .clear],
                                         startPoint: UnitPoint(x: 0.5, y: 1.3), endPoint: .top))
            }
            .containerBackground(for: .widget) { }   // <- empty = transparent
    }
}
```

On **watchOS 10+** this is mandatory: WidgetKit requires `.containerBackground(for: .widget)` on complications. Without it the system supplies a semi-transparent material; the empty builder gives a transparent background so the watch face shows through - what you want for tinted/Lock Screen complications.

For a subtle "lift" behind the glyph, fill a `ContainerRelativeShape` with a faint top-fading gradient (above), so the content reads against busy wallpapers without looking like a solid tile.

## `AccessoryWidgetBackground`

When you *do* want the standard frosted accessory backing, use `AccessoryWidgetBackground()` rather than rolling your own material - it adapts per rendering mode (soft translucent in full-color/accented, black in vibrant):

```swift
ZStack {
    AccessoryWidgetBackground()
    GaugeView(value: entry.value)
}
.containerBackground(for: .widget) { }
```

Most accessory widgets have **no** background - add `AccessoryWidgetBackground` only when a particular style benefits.

## `widgetLabel` - the complication's secondary text

On `accessoryCircular` / `accessoryCorner`, `widgetLabel` provides the curved text/label the watch draws around the complication (bezel / dial). It accepts `Text`, `Gauge`, or `ProgressView` - not arbitrary views:

```swift
ZStack {
    AccessoryWidgetBackground()
    Image(systemName: "cup.and.saucer.fill")
}
.widgetLabel {
    Gauge(value: caffeine, in: 0...max) { Text("Caffeine") }   // bezel gauge on accessoryCorner
}

// String overload:
Image(systemName: "drop.fill").widgetLabel("Hydration")
```

Some faces don't show a label; adapt with `@Environment(\.showsWidgetLabel)`:

```swift
@Environment(\.showsWidgetLabel) var showsWidgetLabel
var body: some View {
    if showsWidgetLabel {
        Image(systemName: "cup.and.saucer.fill").widgetLabel { Text("\(caffeine) mg") }
    } else {
        Gauge(value: caffeine, in: 0...max) { Text("mg") }     // self-contained fallback
    }
}
```

## Gauges, progress, and auto-updating content

Accessory circulars are ideal for `Gauge` and `ProgressView`, which render correctly in tint mode. Prefer the **auto-updating** forms so one entry stays live instead of spamming the timeline:

```swift
Gauge(value: entry.fraction) { Image(systemName: "figure.walk") }
    .gaugeStyle(.accessoryCircularCapacity)

ProgressView(timerInterval: start...end)        // advances on its own
Text(entry.healDate, style: .relative)          // live relative time
```

## Watch complication rendering (full color vs tinted)

On the watch the same accessory families map to complication slots, rendered in a **tint** based on the face. Read `@Environment(\.widgetRenderingMode)` to provide a full-color variant where allowed and a tinted variant otherwise. A real pattern from a shipping app:

```swift
struct WatchComplicationView: View {
    @Environment(\.widgetFamily) var family
    @Environment(\.widgetRenderingMode) var renderingMode

    private var isFullColor: Bool { renderingMode == .fullColor }

    @ViewBuilder var circular: some View {
        if isFullColor {
            Image("ComplicationIcon").resizable().scaledToFit()
        } else {
            // tinted: flatten the multicolor icon to a clean monochrome glyph
            Image("AppGlyph")
                .luminanceToAlpha()
                .background {
                    LinearGradient(colors: [.white.opacity(0.4), .clear],
                                   startPoint: UnitPoint(x: 0.5, y: 1.5), endPoint: .top)
                }
                .widgetAccentable()
                .containerBackground(for: .widget) { }
        }
    }
}
```

The `luminanceToAlpha()` + `widgetAccentable()` combination is the key to a multicolor logo surviving tint mode as a recognizable shape rather than a flat blob - see `rendering-modes-and-tinting.md`.

## Always On and privacy

- **Always On display:** the watch dims and reduces fidelity. Read `@Environment(\.isLuminanceReduced)` and remove time-sensitive detail / dim bright elements when `true`. Relative-time text and `ProgressView`s drop to reduced fidelity automatically.
- **Privacy on the Lock Screen:** mark sensitive views with `.privacySensitive()` so they're redacted (to the placeholder) when the device is locked; the user can also choose to redact all Lock Screen widgets like notifications.

```swift
HStack {
    Image(hero.avatar)                  // always visible
    Text("\(bpm) BPM").privacySensitive()  // redacted when locked
}
```

Use `ViewThatFits` for inline/narrow slots whose width varies by face.

## watchOS specifics: families and recommendations

- `supportedFamilies` differs by platform (system families don't exist on the watch); branch with `#if os(watchOS)`.
- A configurable watch widget/complication has no on-watch editor, so you **must** provide `recommendations()` (preconfigured `AppIntentRecommendation`s) - see `timeline-provider.md` and `relevance-and-smart-stacks.md`.
- Watch complications/widgets have a tight refresh budget (~50 updates/day guidance); watchOS 26 added an **APNs-based** refresh path and the 27 releases add a **watch-connectivity-based** path, on top of timeline reloads. Watch-face complications sit in the top budget tier.

## ClockKit → WidgetKit migration

ClockKit complications are deprecated; build watch complications with WidgetKit accessory widgets and provide a `CLKComplicationWidgetMigrator` so complications already on users' faces upgrade automatically. The 12 ClockKit families collapse to the 4 accessory families. Full details and code in `migration-and-deprecations.md`.

## Checklist

- Accessory backgrounds: **empty** `.containerBackground(for: .widget) { }` = transparent; omitting it = opaque default.
- `AccessoryWidgetBackground()` for the standard frosted backing; `ContainerRelativeShape` to fill the container.
- `widgetLabel` (Text/Gauge/ProgressView) for the curved complication label; `@Environment(\.showsWidgetLabel)` to adapt; `Gauge`/`ProgressView(timerInterval:)` for live circulars.
- Branch on `\.widgetRenderingMode`; `luminanceToAlpha()` + `widgetAccentable()` for multicolor glyphs in tint mode.
- `\.isLuminanceReduced` for Always On; `.privacySensitive()` for Lock Screen data; provide `recommendations()` on watchOS.
- Build complications with WidgetKit + a migrator, not ClockKit.
